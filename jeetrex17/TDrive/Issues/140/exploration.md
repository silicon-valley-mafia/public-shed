# TDrive #140 — "Forgot encryption password": multi-agent exploration

**Date:** 2026-11-23 · **Base:** `jeetrex17/TDrive` master `52bc8de4` · **Requested by:** Deepesh J
**Method:** Round 0 refreshed the known facts. Then came five sets, each run as five rounds (R1 = same goal for all agents, R2–R3 = directed, R4–R5 = counter/falsify). The same six agents ran every round: claudesonnet-1, claudesonnet-2, claudefable, gptluna, gptterra, gptAstra.
**Tags:** VERIFIED = read in source · REASONED = inferred · MEASURED = run in a sandbox.

| Set | Topic | File | Commit |
|---|---|---|---|
| A | Round 0 + rescuing the already-locked-out | `sets/set-a-rescue.md` | `94b1c2da` |
| B | Recovery-key design | `sets/set-b-recovery-key.md` | `439467bf` |
| C | Threat model, old-wrap retirement, hints, policy | `sets/set-c-threats-retirement.md` | `bb43c28f` |
| D | Implementation plan + UX copy | `sets/set-d-impl-ux.md` | `4a739f60` |
| E | Upstream comment + disclosure | `sets/set-e-upstream-comment.md` | this push |

---

## 1. Ground truth (VERIFIED)

- **Key structure.**
  - A random 32-byte master key is wrapped with XChaCha20-Poly1305 under an Argon2id KEK (64 MiB, t=3, p=4). The wrap is a 72-byte blob.
  - A key-check `tdrive-key-check-v1` is sealed under the master key.
  - Per-file subkeys are derived with HKDF-SHA256 from the master key, using the salt in each file's TDE1 header and info `tdrive/file/v1`.
- **Config storage.** Config is an `encfg` control message in the personal channel: `TDX1|t=encfg|salt|kdf|wrap|check|hint|cv`, with the **hint in plaintext**. It is published before it is stored locally.
- **Password changes.**
  - `ChangePassword` **re-wraps the same master key**. Only `CreatePasswordContext` mints a new one.
  - **Old `encfg` messages are never deleted**, so every password ever used still unwraps today's key.
- **The no-reset decision.**
  - Jeet wrote *"There is intentionally no "forgot password" reset"* in `encryption-settings.ts` on 2026-05-01 (`29245079`).
  - v1.0.0 (2026-05-09) notes say *"There is no password reset."*
  - **Jeet himself opened #140 on 2026-10-08**, suggesting Telegram/SMS. So he reopened the question.
- **Biometrics.**
  - PR #153 merged 2026-10-13, and biometrics first shipped in **v2.3.0 (2026-11-07)**.
  - Released builds support only Windows Hello (which accepts the PIN) and Android `BIOMETRIC_STRONG`.
  - macOS is ad-hoc signed, so Touch ID fails. There is no iOS release.
  - A password change deletes the enrollment, and so does logout. Locking keeps it.
- **Session key.** It is held in RAM only, until the app exits or the vault is locked. There is no idle auto-lock.
- **Plaintext metadata.** File names, sizes, the folder tree, and hints are all plaintext.

## 2. Why Telegram/SMS can't do it

A Telegram login or an SMS code proves control of the account. It cannot rebuild a **random** key that was never derived from it. Recovery needs a second copy of the key that was **saved beforehand**, and whoever holds that copy joins the trust boundary.

Options considered and rejected:
- **Mini App CloudStorage:** bot-scoped.
- **Passport `secure_settings`:** overwriting it breaks the user's real Passport.
- **Saved Messages:** same account as the ciphertext.
- **Bots:** the operator becomes part of the trust boundary.

TDrive has no server.

## 3. Rescue for a user who is already locked out (di-kun)

**Nothing in TDrive today rescues a user who has no unlocked session and no valid enrollment.** di-kun reported the problem on 2026-10-14, before any release had biometrics, so the device route almost certainly doesn't apply to di-kun.

What would work, in order:
1. A session that is still open and unlocked: download everything.
2. A Windows or Android enrollment that hasn't seen a password change since.
3. *A remembered OLD password* — this needs new code (see §5, PR 2).
4. Hints, which are in the channel history.
5. Guessing candidate passwords:

| Hardware | Rate (Argon2id 64MiB/t3/p4) |
|---|---|
| Single Xeon vCPU | **152.4 ms per guess (MEASURED)** |
| Laptop | ≈ 9–33 guesses/s (≈ 13/s typical) |
| RTX 4090 | ≈ 1,700–1,800 guesses/s |

## 4. Design choice: recovery key = the master key (design X)

- **Choice.** All six agents picked X in Set D, independently. Set B left X and Y' both viable.
- **What X is.** The master key itself, shown as 52 Crockford base32 characters in 13 groups of 4, with zero pad bits.
- **Why it's cheap.**
  - It is validated by the existing `VerifyKeyCheck`.
  - No wire change, no schema change, no rescan.
  - It works on existing vaults and on all four released GUIs (Windows, Android, macOS arm64, Linux AppImage). No CLI change is needed.
- **Costs.**
  - It can never be rotated. Anyone holding it can decrypt everything.
  - It needs a carve-out in two `encryption.md` sentences: "keys never enter the frontend bindings" and "the password remains mandatory for … recovery".
- **Y' (rejected).** A 128-bit key, 28 characters, stored in extra `encfg` fields. Rejected because:
  - `configOp` copies only known fields, so an old client's password change silently drops the recovery wrap.
  - It would need new columns.
  - Its rotation is cosmetic, because old wraps persist.
- **Test vectors (MEASURED, cross-checked by two agents and by me):**
  - `0x00..0x1f` → `000G40R40M30E209185GR38E1W8124GK2GAHC5RR34D1P70X3RFG`
  - `0xff×32` → 51×`Z` + `G`
  - `0x42×32` → `89144GJ2`×6 + `8910`

## 5. Implementation plan

| # | PR | Prod / test / FE LOC | Notes |
|---|---|---|---|
| 0 | **Orphan-key guard** | ~30 / ~150 / ~10 | `CreatePasswordContext` mints a new key if all `encfg` messages are gone even when encrypted files exist. Guard on `files`/`file_revisions.encrypted=1` after an authoritative sync, and fail closed. A #194-sized change. |
| 1 | **Recovery key (X)** | ~290 / ~340 / ~330 | See the details below the table. |
| 2 | **Explicit "Try an earlier password"** | ~150 / ~240 / ~100 | See the details below the table. |
| 3 | **Set a new password from a fresh enrolled-device unlock** | ~185–350 total | Windows/Android only. Windows Hello accepts a PIN. |
| 4 | **Stop publishing new hints** | ~60–120 | Old hints stay in history. The unlock modal hides "Hint:" when the hint is empty. |
| 5 | **New-password policy** | ~90–180 | At least 12 code points plus feedback. A generated 5-word EFF passphrase is 64.6 bits, while 4 words is 51.7 bits and fails the target. |

**PR 1 details:**
- Reveal requires password re-entry and never uses `LoadedMasterKey` alone.
- Reset order: verify the key → `buildConfig` (same key) → `publishConfig` → `StoreMasterKey`, then a **best-effort** enrollment delete. A failure there returns `biometric_cleanup_required`.
- `OperationResult` cannot carry a warning, so the result needs an envelope type. Add `invalid_recovery_key` to `OPERATION_ERROR_CODES`.
- Add an entry point in the photo-preview unlock card too.
- Regenerate bindings (wails3 `v3.0.0-beta.22`).

**PR 2 details:**
- History comes from `replay_log`, newest first, skipping tampered rows, capped at N=8.
- A match must pass `VerifyKeyCheck` against the **current** check.
- Re-wrapping under a new password is forced, within one call.
- If publishing fails, the vault stays locked.

**Out of #140** (separate issues):
- Keep enrollments across password changes. `encryption.md` and the tests promise deletion.
- Retire old `encfg` messages (forward-looking hygiene only).
- Master-key rotation (≈400–700 LOC, config v2).
- Encrypt `session.json` (raw gotd session, 0600).
- Restore when the config is gone, by authenticating the key against a surviving file via `NewRandomAccessDecryptor` (needs the stored size; one file proves the key for that file only).

## 6. Threat model (Set C)

Threats in order:
1. Local session theft (`session.json`).
2. Cracking a weak current password. The minimum is 8 **bytes**, and there is no strength meter.
3. **Old-wrap retention.**
4. Plaintext hints. NIST SP 800-63B-4 says hints "SHALL NOT" be accessible to an unauthenticated claimant.
5. A leaked recovery key, or abuse of a reset (both proposed features).

Retirement rule: after the new config round-trips, delete **all** older `encfg` messages. Use a new delete path filtered on `op_type=encfg`. This is hygiene, not revocation.

Example of the residual risk, at 100 GPUs and 170k guesses/s:

| Attacker copied the channel… | Expected time to crack |
|---|---|
| After retirement | ≈ 2.65 million years |
| Before retirement, via an old 8-lowercase-letter password | ≈ 7.1 days |

## 7. Upstream plan (Set E)

**Maintainer calibration:**
- Only one external PR has ever been merged: #194, +80/−25.
- A 148-line design PR, #196, was closed in 13 minutes.
- CONTRIBUTING: *"Discuss substantial behavior or architecture changes before implementing them."*
- No reply yet on #195 (since 2026-11-22), #140, or #193.

**Public comment for #140:** the agents' 199-word version (6/6 sign-off after two BLOCK fixes) left out the old-password property. It was superseded by **v6**, which adds the property and "Try an earlier password" following the decision below. Both versions are in `sets/set-e-upstream-comment.md`.

**Old-password property — decision (2026-11-23, Deepesh): mention it publicly in #140.**
- The six agents had recommended private-only, because TDrive's docs publish the building blocks ("re-wraps the same master key") but not the implication.
- Deepesh's reasoning: reading the channel already requires the user's own Telegram login. TDrive runs locally with the user's own API ID, and the drive is a private channel the user owns. So the audience is limited to people who can already sign into the account (or Telegram itself).
- Correction recorded alongside the decision: old passwords still working is **not** inherent. Files are encrypted with the master key, not the password, so old files open fine with a new password. Old passwords work only because older config messages are never deleted. Deleting them after a successful change would stop that going forward, without touching any file. Copies someone already took stay usable.

## 8. Calibrations (what the falsify rounds caught)

- "61 H/s per GPU" was withdrawn; the figure is ~1,700–1,800/s. The "4 guesses/s" was a design target, not a measurement.
- The Privacy Policy §10.2 48-hour retention applies to supergroups, so retention for channels is **unknown**.
- "A new op type needs a rescan" is true, which is why X (or extra fields) avoids it.
- "Biometric-only reset" isn't possible on Windows, because Hello accepts the PIN.
- Delete-enrollment-first on a recovery reset was falsified. An offline failure leaves the user strictly worse off.
- "Keep enrollments" (Option B) was falsified: doc promise, race check, and tests.
- Three UX-copy rounds introduced falsehoods ("replace an earlier password", "only the new password works"). Every string must be checked against the old-wrap fact.
- An agent cited a **fork** (`ndgthuan/tdrive`) as the source. Only `jeetrex17/TDrive` is authoritative.
