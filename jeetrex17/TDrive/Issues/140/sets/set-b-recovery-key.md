# Set B — Designing a recovery key for future users

TDrive master `52bc8de4`, 2026-11-23. Five rounds, six agents per round.

## R1 — same goal: pick a recovery mechanism

- Five of six agents picked **R3: a recovery code wraps the same master key, and the wrap is published to the channel.** gptluna picked **R2: an offline kit file**, to avoid a new op type.
- All six agreed on these points:
  - **HKDF-SHA256, not Argon2id**, for a random code. Argon2id stays for human passwords.
  - **Domain-separated** info and AAD.
  - Verify the result with the existing `VerifyKeyCheck`.
  - Users opt in while the vault is unlocked. No re-encryption is needed.
  - **Recovery must never mint a new master key.**
- **Biggest risk (all six):** a leaked code plus channel access unlocks everything, permanently. Rotation is only logical, because old wraps persist. True revocation means a new master key and re-encrypting every file.
- Proposed entropy varied: 128, 130, 160 or 256 bits. Settled in R2 and R4.

## R2 — directed

- **Entropy and encoding (sonnet-1):**
  - An RTX 4090 computes HMAC-SHA256 at 4,333.9 MH/s (govcracker). That makes HKDF plus an AEAD check at most about 3×10⁹ guesses/s per GPU.
  - Searching half of 2^128 takes 1.8×10²¹ GPU-years, or 1.8×10¹⁵ years on a million GPUs.
  - **128 bits is enough.** Argon2id on the code would add only 20–30 bits to a search that is already infeasible.
  - Crockford Base32 makes that 26 characters. A 12-word BIP39 phrase is equally strong but has only a 4-bit checksum.
- **Wire format (sonnet-2):**
  - Parse reads these keys generically for every op: `t sz ts enc ev psz` (and `rend`, which only `part` ops may carry).
  - A proposed `encwrap` op is about 190 characters worst case.
  - Control ops are sent as **text messages** (`SendControl`), not captions.
  - Putting the wrap in an extra `encfg` field has a cost: `configOp` builds a fixed struct, so **an old client's password change drops that field.**
- **Rescan problem (fable):**
  - `currentSchemaVersion = 14`. The only precedent for forcing a rescan is `flagTruncatedScansForRebuild` (v<9), which applies only to matching channels.
  - `applyDifference` and `filterHistoryPage` drop ids at or below the watermark.
  - `tgclient.SearchChannelMessages` (MessagesSearch, filter Empty) exists but has no caller.
  - Cost for 100k messages: forced rescan ≈ 2,001 `getHistory` calls; targeted search ≈ 1 call.
- **Offline kit (luna):**
  - 1Password's kit holds sign-in address, email, Secret Key, a blank password field and a QR code.
  - Proton's holds a 12-word phrase.
  - Cryptomator's recovery key is "the decrypted masterkey in human-readable form".
  - A kit holding code + wrap + salt survives losing the channel, but it is then **a complete key**.
- **Mixed versions (terra):**
  - An old client's password change cannot break a separate recovery op, because the master key never rotates.
  - The latest generation is the one with the highest `msg_id`.
  - After the code is used, require a fresh code. Publish it before retiring the old one.
- **Implementation map (astra):**
  - Add `WrapMasterKeyAAD`/`UnwrapMasterKeyAAD` and `DeriveRecoveryKEK`.
  - Add `OpEncRecovery` and an `encryption_recovery` table (schema v14→15).
  - Add `CreateRecoveryCode` (needs the current password) and `RecoverWithCode`.
  - Add app bindings. **CI pins `wails3@v3.0.0-beta.22` and fails on stale `frontend/bindings`.**
  - Roughly 450–700 lines including tests.

## R3 — directed

- **Design X vs design Y (sonnet-1).** X: the code **is** the master key, Cryptomator-style — 52 characters or 24 words, validated with `VerifyKeyCheck`, **no new op and no schema change.** Y: a 128-bit code wraps the key. Y's rotation adds no real security, because old wraps persist. Verdict: **ship X first.**
- **Test plan (sonnet-2).** Style matches `TestChangePasswordKeepsMasterKey`. Proposed tests:
  - `TestRecoverKeepsMasterKey` (encrypt a fixture, recover, decrypt it)
  - `TestRecoverWrongCodeNoStateChange`
  - `TestRecoverTamperedWrapRejected`
  - `TestRecoverPublishFailureKeepsOldPassword`
  - `TestRecoverAfterOldClientChangePassword`
  - `TestRecoverOldWrapPersistsInHistory` (documents the residual risk)
  - `TestRecoverTryOlderPasswordFallback`
  - `TestResetFromBiometricKeepsMasterKey`
  - Each test carries a `// guard:` line naming the line it protects, so a mutation check can prove the test fails without it.
- **Search (fable).**
  - Control ops send the header only, with no second line.
  - `ExtractHeaderLine` stops at the first `\n`.
  - Telegram search tokenization is undocumented and assumed to be whole-word. Put a second-line marker (`TDrive tdxencwrap`) on the message and treat any search hit as a hint to be re-parsed.
- **UX copy (luna):** first draft; corrected in R4/R5.
- **Reset from biometrics (terra).** Validate the new password, then: fresh `provider.Unlock` → re-take the gate → check scope + `VerifyKeyCheck` → `buildConfig` with the same key → `publishConfig` → `StoreMasterKey` → **then** delete and re-enroll biometrics. Backgrounding the app on Android cancels the attempt.
- **CLI (astra).**
  - Today the CLI has `tdrive vault status|unlock|lock`, an `unlock` alias and `--password-stdin`. **There is no change-password command.**
  - `tdrive vault recover` and `recovery-key create` would go next to `vaultUnlock`.
  - **On Linux the recovery key would be the only recovery path.**
  - Docs to update: README "There is no password reset. The optional hint cannot decrypt content." and encryption.md "The password remains mandatory for setup, recovery, password changes and enrollment on another device."

## R4 — counter/falsify

- **X weakened (sonnet-1).**
  - encryption.md says *"Passwords, keys and native signatures never enter the frontend bindings or runtime events."*
  - `CreatePasswordContext` already holds the master key, so X can be shown at creation for free.
  - If **all** `encfg` messages are gone, X has no key-check to validate against.
- **sonnet-2 retracted its own test #5.** `buildConfigWithKDF` takes no existing config, and `configOp` copies only salt/kdf/wrap/check/hint/cv.
- **Ordering confirmed.** Configs replay by `msg_id` ascending with last-wins upsert. Old headers stay in `replay_log`, so a device that hasn't synced still accepts the old password.
- **128 bits SURVIVES (fable)**, provided each wrap has its own salt:
  - Multi-target without salt: about 2^108 work against 10^6 users. With salt: 2^128 per pair.
  - Grover: NIST says "AES 128 will remain secure for decades"; ETSI estimates about 2^87 gates.
  - RFC 5869 sets no minimum IKM length.
  - Final: **128 bits.**
- **Copy fixes (luna):**
  - Call it a **"recovery key"**, as Cryptomator does.
  - Drop "we": TDrive has no server.
  - Be honest about biometric limits.
  - Question whether to offer "Try an older password" (settled in R5).
- **Reset race (terra).** Re-checking the scope does not fully close the race: sync can replace the config after the last check. `ChangeEncryptionPassword` does **not** eject the mount; mounts hold their own key lease. CLI claims confirmed.
- **Search marker (astra).**
  - `HashHeader` and `first_seen_hash` cover line one only, so a second-line marker is harmless.
  - Schema v14 and the `flagTruncatedScansForRebuild` precedent confirmed.
  - `SearchChannelMessages` has no sync caller.

## R5 — counter/falsify (final)

- **X vs Y are equal on secret exposure (sonnet-1).** Passwords already cross the bindings (`CreateEncryptionPassword(password, hint)` and friends), and that docs sentence sits in the biometric section. Objection dropped.
- **⚠ NEW, independent hazard (sonnet-1, VERIFIED).** If there is no local row **and** no `encfg` message on the channel (a fresh install after those messages were deleted), `EnsurePersonalConfig` returns exists=false. `CreatePasswordContext` then calls `NewMasterKey()` with **no check for existing encrypted files**, which orphans the old ciphertext.
- **Extra `encfg` field is SAFE for mixed versions (sonnet-2, VERIFIED).** This decides the design:
  - Parse reads unknown keys into a `kv` map and ignores them. The code comment says encryption keys are *"additive: older clients ignore them"*.
  - Rebuild does no strict per-field check. The real breaker is `cv`: `validateEncryptionConfig` rejects any value other than 1 and **fails the whole vault closed**. So keep `cv` at 1.
  - An old client's password change drops the field, so new clients must carry it forward. Removing it needs an explicit empty value.
  - **⇒ An extra `encfg` field plus carry-forward avoids the rescan problem completely**, because the field rides on messages every client already projects.
- **Final crypto spec (fable):**
  - Use 2 Base32 check characters taken from SHA-256, not Crockford's mod-37 symbol. Its values `* ~ $ = U` are awkward to type and unsafe in URLs/QR.
  - Use **full HKDF extract-and-expand.** Expand-only needs a key of at least 32 bytes, and the code is 16.
  - A public salt is fine.
  - Spec:
    1. Key = 16 random bytes → 26 Crockford Base32 characters.
    2. Check = 2 characters from SHA-256(`tdrive/recovery-check/v1` ‖ key). Display 28 characters as 7 groups of 4. The check is never published.
    3. KEK = HKDF-SHA256(key, 16-byte salt, `tdrive/recovery-kek/v1`, 32 bytes).
    4. Wrap = XChaCha20-Poly1305 with AAD = label ‖ recovery id ‖ salt.
    5. Decoder: uppercase input, I/L→1, O→0, strip hyphens.
- **Copy (luna):**
  - Biometrics in RELEASED builds: Windows Hello and Android only. Not macOS or Linux; there is no iOS release.
  - Hiding "try an older password" would be **security by obscurity that only hurts legitimate users**. Suggested string: *"Current password not working? Try an earlier vault password. This may unlock your vault; it doesn't reset your password."*
- **The race is PRE-EXISTING (terra).** `ChangePassword` already has the same read-to-publish window, and `EmitAndProjectContext` does not take the sync gate. It is not new to a reset feature.
- **Second-line marker is safe (astra).** No existing control op uses multi-line text, and loose-file detection needs media. Telegram has no control-only search filter. Use the marker only as a hint and re-validate line one.

## SET B — conclusions

1. **Two designs are viable, and neither needs a new op type.**
   - **X:** the recovery key IS the master key (52 characters / 24 words). Zero wire change. Can never be rotated.
   - **Y':** a 128-bit recovery key wraps the master key. The wrap goes in **extra `encfg` fields** (keep `cv=1`) and new clients carry it forward. Short (28 characters), and "rotation" is honest only for cooperating clients.
2. A separate op type is the worst option: it forces a rescan or search because of the watermark.
3. Crypto: 128-bit key, HKDF extract+expand, AAD-bound XChaCha20-Poly1305 wrap, 2-character SHA-256 check.
4. Reset from biometrics: publish first, delete the enrollment after. The race window is pre-existing.
5. Fix independently of #140: **`CreatePasswordContext` can mint a new key over orphaned ciphertext when all `encfg` messages are gone.**
6. On Linux and macOS release builds, the recovery key is the only future recovery path.
