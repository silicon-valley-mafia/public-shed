# Set D — Implementation plan + UX copy

TDrive master `52bc8de4`, 2026-11-23. Five rounds, six agents per round (claudesonnet-1, claudesonnet-2, claudefable, gptluna, gptterra, gptAstra).
**VERIFIED** = read in source. **REASONED** = inferred. **MEASURED** = run in a sandbox.

## R1 — same goal: phased PR plan + UX map

- **All six independently picked design X first**: the recovery key IS the master key, 52 Crockford characters, no wire or schema change.
  - Y' was rejected because `configOp` copies only known fields, so any old client's password change silently drops the recovery wrap.
  - Y' would also need new columns in the `encryption` table.
  - Y' revocation is mostly cosmetic, because old wraps persist.
- **UI map (VERIFIED).** There is no router. The vault screens are modals mounted in `frontend/src/ui/app/FeatureLayer.svelte`. `requireEncryptionPassword` in `frontend/src/modules/encryption.ts` chooses between setup and unlock.

| Screen | Component (`frontend/src/ui/modals/`) | Controller (`frontend/src/modules/modals/`) |
|---|---|---|
| Create | `EncryptionSetupModal.svelte` | `encryption-setup.ts` `submitEncryptionSetup` |
| Unlock | `EncryptionPasswordModal.svelte` | `encryption-password.ts` `submitEncryptionPassword` |
| Unlock inline (photo preview) | `ui/preview/PreviewModal.svelte` `#preview-locked` | `preview-unlock-card.ts` |
| Change password / device unlock | `EncryptionSettingsModal.svelte` | `encryption-settings.ts` `submitEncryptionSettings` |

- The service lives at `backend/services/encryption/service.go`, not `backend/encryption/`.
- **⚠ Key finding (VERIFIED by me).** `encryption-settings.ts` opens with:
  > *"There is intentionally no "forgot password" reset; without the current password encrypted files cannot be recovered."*

  Jeet added it in commit `29245079` on 2026-05-01, then opened #140 himself on 2026-10-08. **He reopened the question himself.** (R5 astra)
- PR-order proposals disagreed: orphan guard first (sonnet-1, terra, astra), password policy first (sonnet-2), hints first (fable). Settled in R5.

## R2 — directed

- **Biometric scope (sonnet-1).**
  - `Scope.ID()` = digest over account + channel only, so Enroll, Unlock and Delete all hit the same item across password changes.
  - `ConfigHash` is enforced inside the item. Windows binds it in `readRecord`, `windowsAAD` and the HKDF info. Android compares it in `status`/`authenticate`. Apple checks it via `DecodeSecret`.
  - **Enroll always prompts on all three platforms**, so re-enrollment can never be silent. Delete is silent.
  - The key-check is re-sealed with a fresh nonce on every change, so it is not a stable identity.
- **X vs the docs (sonnet-2).**
  - `internal/app/biometric.go`: *"keys never enter Wails"*.
  - `docs/architecture/encryption.md`: *"Passwords, keys and native signatures never enter the frontend bindings or runtime events"* and *"The password remains mandatory for setup, recovery, password changes and enrollment on another device."*
  - Passwords already cross the bridge (`CreateEncryptionPassword`, `UseEncryptionPassword`, `ChangeEncryptionPassword`, `EnableBiometricUnlock`), but X still needs an **explicit docs carve-out**.
  - `OperationResult` has no data payload. Wrapper types exist (`PreviewResult`, `DownloadResult`, `UploadResult`).
  - **Crockford beats BIP39**: no wordlist, no dependency, and it doesn't look like a crypto-wallet seed.
  - **Never reveal from `LoadedMasterKey` alone**: a biometric-unlocked session would leak the key with no password.
- **"Try an earlier password": automatic or explicit? (fable)** Verdict: **EXPLICIT.**
  - Automatic fallback gives an attacker nothing new.
  - But every typo would cost up to 9 derivations, and users would never learn their current password.
  - No salt dedupe is possible: every change draws a fresh salt.

| Device | Per derivation | 9 derivations (current + 8 older) |
|---|---|---|
| Laptop, 4 threads | 75 ms | 675 ms |
| Xeon vCPU, 1 thread | 152.4 ms | 1,371.6 ms |
| iPhone XS-class (64 MB/t3/p1 vendor figure) | 410 ms | 3.69 s |

- **Orphan guard (terra), schema VERIFIED.**
  - `channels(channel_id, initial_sync_done, needs_projection_rebuild)`.
  - `files.encrypted` and `file_revisions.encrypted`, both with `channel_id`.
  - `EnsureAuthoritative` only does an incremental refresh under `InitialSyncDone && !NeedsProjectionRebuild`.
  - `CreatePasswordContext` → `EnsurePersonalConfig` → refresh → rebuild, and only then returns exists=false.
  - About 25–35 production LOC plus 100–180 test LOC.
  - README already warns against deleting TDX messages.
- **Maintainer calibration (astra).**

| PR | Author | +/− | Files | Open → merge |
|---|---|---|---|---|
| #198 | Jeet | 822/173 | 22 | 11:48 |
| #199 | Jeet | 687/20 | 13 | 11:24 |
| #197 | Jeet | 445/19 | 15 | 0:30 |
| **#194** | **NithishKumar2772** | **80/25** | **5** | **4:24** |
| #195 | Jeet | 499/10 | 19 | 0:19 |

  - **#194 is the only external PR ever merged.**
  - The same contributor's 148-line design-doc PR #196 was **closed in 13m25s**: *"I'm closing this PR and keeping the discussion in #193"*. #193 still has 0 comments.
  - CONTRIBUTING.md: *"Discuss substantial behavior or architecture changes before implementing them."* It also asks for squash-merge, titles ≤72 characters, and a fixed PR template (Description / Related Issue / Motivation and Context).

## R3 — directed

- **Enrollment survival (sonnet-1).** Proposed Option B (stable hash) with read-side dual-accept: no rewrite, no extra prompts, about 75 LOC. Falsified in R4/R5.
- **X API (sonnet-2).**
  - `crypto.EncodeRecoveryKey` / `DecodeRecoveryKey` and `ErrInvalidRecoveryKey`.
  - `Service.RevealRecoveryKey(ctx, password)`: unwraps, so it works while locked.
  - `VerifyRecoveryReset` (no Argon2) and `ResetPasswordWithRecoveryKey`.
  - Error code `OperationCodeInvalidRecoveryKey = "invalid_recovery_key"` in `operation.go` `operationErrorCode`.
  - The reveal can follow `CreateEncryptionPassword` with **no signature change**: the UI calls reveal with the just-typed password, at the cost of one extra Argon2 (~250 ms).
  - Estimate: about 220 production / 370 test Go LOC.
- **Test plan (fable).** Uses `newTestService`, the `svc.deriveKEK` and `svc.personalChannelID` overrides, and the `TestChangePasswordKeepsMasterKey` pattern.
  - Ten tests, each with a `// guard:` line naming the production line it protects.
  - Mutants: dropping `VerifyKeyCheck` is killed by the wrong-key tests (no publish). Swapping publish and store is killed by the publish-failure test. A reveal that reads `LoadedMasterKey` is killed by the decoy test.
  - Frontend tests go in `frontend/src/ui/modals/encryption-modals.dom.test.ts`.
- **Vectors (MEASURED, re-run locally by me):**
  - `0x00..0x1f` → `000G40R40M30E209185GR38E1W8124GK2GAHC5RR34D1P70X3RFG`
  - `0xff×32` → 51×`Z` + `G`
  - `0x42×32` → `89144GJ2`×6 + `8910`
  - All three are 52 characters. The last character is always `0` or `G`, because the 4 pad bits are zero.
- **Rescue after config loss (terra).**
  - With zero `encfg` messages, X can't use `VerifyKeyCheck`.
  - It can authenticate against a surviving encrypted file instead, via `crypto.NewRandomAccessDecryptor`. That function authenticates the **final** chunk on open.
  - Then republish a fresh `encfg` with the **same** key.
  - A remembered password alone can't do this: the wrap is gone.
- **First proposal (astra).** Lead with the recovery key. It covers all four released platforms and doesn't make retired passwords an official credential.

## R4 — counter/falsify

- **Reset order (sonnet-1). The R3 proposal is FALSIFIED** (delete enrollment → reset).
  - `ChangeEncryptionPassword`'s delete-first order is paired with `VerifyBiometricPasswordChange`, which checks both passwords. A recovery reset has no old password.
  - With delete-first, an offline publish failure leaves the user **strictly worse off**: still locked out, and the biometric path is gone too.
  - **Correct order:** verify → publish → `StoreMasterKey` → best-effort `deleteBiometricEnrollment`.
  - The old enrollment is already inert, because its `ConfigHash` no longer matches.
  - Holding the lifecycle gate across Argon2 is fine (55 s timeout; Use and Change already do it). Better still: derive outside the gate, then re-check under it.
- **Fixture check (sonnet-2).** Mostly VERIFIED, with corrections:
  - `newTestService` returns `(*Service, *sql.DB)`.
  - The publisher field is `emitOp`.
  - The encryption package uses `masterKeySize`; `masterKeyLen` is the crypto package's name.
  - The decrypt pattern is in `biometric_test.go`.
  - Call `svc.Clear()` before failing-emitter tests.
  - No new seams are needed.
- **Orphan rescue (fable): SURVIVES, with corrections.**
  - `NewRandomAccessDecryptor(ctx, source, storedSize, masterKey)` also needs the **stored size**.
  - Multipart files form one TDE1 stream, so only part 1 has a header.
  - There is no AAD and no file-id binding.
  - Fetches are whole blocks of up to `RangeReadMaxBytes` = 1 MiB.
  - One file proves the key only for that file.
  - Estimate: about 120–180 production LOC, reusing `mountcontent.Opener`.
- **"Earlier password conflicts with policy" (fable): NO written policy found.**
  - It exposes behaviour the cryptography already has; it isn't a new capability.
  - 1Password: *"If you've ever changed your account password, try previous ones."* Caveat: there, it only unlocks a local copy.
- **Option B falsified (terra).**
  - `encryption.md` explicitly promises *"native deletion must succeed before the password change proceeds"*.
  - Dual-accept dies after any synced password change.
  - A stable hash removes the mid-prompt rewrap check (`TestBiometricLateCallbackRejectsRewrappedConfig`).
- **X claims (astra).**
  - Zero wire change: SURVIVES.
  - All four platforms ship a GUI (Windows installer/portable/CLI, Android APK, macOS arm64, **Linux AppImage + CLI**): SURVIVES. So the **CLI is not required**.
  - Telegram/SMS escrow: identity alone can't decrypt anything. A key stored in Saved Messages sits in the same Telegram account as the ciphertext. A bot puts its operator inside the trust boundary.
- **Copy (luna v4).** Introduced new falsehoods ("use the recovery key to replace an earlier password"; the key only "sets a new password"). Fixed in R5.

## R5 — counter/falsify (final)

- **Keep enrollments: OUT of #140 (sonnet-1 concedes).** Tests `TestBiometricPasswordChangeDeletesEnrollment` and `TestBiometricLateCallbackRejectsRewrappedConfig` encode the current semantics. Deleting the enrollment is the only *local* revocation when the master key never rotates. File it separately.
- **Docs amendment for the recovery path (sonnet-1):**
  > *"Recovery-key reset has no old password to verify, so it verifies the recovery key and publishes the new configuration first, then attempts native deletion; the prior enrollment is bound to the superseded configuration and cannot unlock, and a failed deletion is reported as `biometric_cleanup_required` and stays retryable through Disable biometric unlock."*
- **PR-1 spec (sonnet-2), VERIFIED gap:**
  - `OperationResult` can't carry success plus a warning. `normalizeOperationResult` drops `error` on `ok:true`, and `types.ts` types success as `error?: never`.
  - Fix: a `RecoveryResetResult{Result, Warning *OperationError}` envelope.
  - `invalid_recovery_key` must be added to `OPERATION_ERROR_CODES`, a `Record<…,true>`, or the build breaks.
  - Put the new methods behind a type-asserted `recoveryEncryptionService` interface, so existing fakes keep compiling.
  - Revised estimate: about 290 production Go / 340 test / 330 frontend LOC.
- **PR-2 spec (fable).**
  - `readEncryptionReplay` LEFT JOINs `replay_log_tamper` and returns every row. The abort happens in `RebuildEncryptionConfigFromReplay`, so a history lister can `continue` past bad rows.
  - `ListEncryptionConfigHistory` returns rows newest-first.
  - `ResetWithEarlierPassword(ctx, earlier, new)` is one call: the forced re-wrap is structural, and no key is held between steps.
  - Cap N = 8.
  - **If publish fails, stay locked**. A session-only unlock would make a retired password a repeatable credential.
  - Biometric cleanup must use the passwordless `deleteBiometricEnrollment`, not `removeBiometricEnrollment(…, current, new, true)`.
  - Estimate: about 150 production / 240 test / 100 frontend LOC.
- **Order attack (terra).**
  - Put the guard first, since reinstall and reset flows make reaching the orphan path more likely.
  - Device-method reset is **not** redundant: it helps people who enrolled but never saved a key.
  - Stopping hints doesn't break unlock: the modal renders "Hint:" only when the hint is non-empty.
- **Collision check (astra).**
  - Master is still `52bc8de4`.
  - The only open PR is #200, which preserves the `encrypt` flag on retry and doesn't touch vault code.
  - `encryption.md` has no "forgotten password" section.
  - Jeet has never replied on #140 or #193.

## SET D — conclusions

1. **Design X, as an explicit opt-in to see the key.** Every vault already has a recovery key, because it IS the master key. The UI only reveals it after password re-entry.
2. **Final PR order:**

| # | PR | Prod / test / FE LOC | Why here |
|---|---|---|---|
| 0 | Orphan-key guard in `CreatePasswordContext` | ~30 / ~150 / ~10 | Data-loss safety. Tiny, #194-sized, and a good first external PR. |
| 1 | **Recovery key (X)**: reveal + reset, publish first | ~290 / ~340 / ~330 | The actual #140 fix for future users, on all four platforms. |
| 2 | Explicit "Try an earlier password" + forced new password | ~150 / ~240 / ~100 | The only path that can help an already-locked-out user. |
| 3 | Set a new password from a fresh enrolled-device unlock | ~185–350 total | Helps people who enrolled but never saved a key. Windows Hello may accept a PIN. |
| 4 | Stop publishing *new* hints | ~60–120 | Privacy. Old hints stay in history. |
| 5 | Password policy for new passwords (≥12 code points + feedback) | ~90–180 | Hardening. |

3. **Out of #140** (separate issues): keep enrollments across password changes · retire old `encfg` messages · master-key rotation · encrypt `session.json` · restore when the config is gone, verified against a surviving encrypted file (after PR 1).
4. **Docs:**
   - Amend the two `encryption.md` sentences, plus the recovery-reset deletion sentence above.
   - Replace the README line *"There is no password reset. The optional hint cannot decrypt content."*
   - Update the header comments in `encryption-settings.ts` and `encryption-setup.ts`.
5. **Final UX copy.** Starts from luna's R5 table. I made 5 edits: removed internal jargon ("encfg wraps"), merged the duplicate device/biometrics option, set confirm to 3 random groups, made "export" point at a still-unlocked device, and removed the "replace" wording.

| id | string |
|---|---|
| forgot.title | Forgot your password? |
| forgot.option.recovery | Use recovery key |
| forgot.option.earlier | Try an earlier password |
| forgot.option.device | Use {deviceMethod} *(only when available and enrolled; falls back to "biometrics")* |
| forgot.device.hello_note | Windows Hello may ask for your PIN. *(Windows Hello only)* |
| forgot.export | Still unlocked on another device? Download your files there first. |
| forgot.last | TDrive has no server that can reset your password. Without your recovery key, an earlier password or an enrolled device unlock, encrypted files can't be decrypted. |
| recovery.create.title | Save your recovery key |
| recovery.create.body | Every vault has a recovery key. It is the master key for your encrypted files. |
| recovery.warning | Anyone with this key can decrypt every encrypted file, no password needed. Changing your password doesn't change it. Keep it private. |
| recovery.confirm | Enter groups {a}, {b} and {c} to confirm you saved it. |
| recovery.skip | Skip for now. You can show the key later in Settings after entering your password. |
| settings.show.title | Show recovery key |
| settings.show.body | Enter your password to reveal it. |
| recovery.use.body | Enter your 52-character recovery key to set a new password. |
| earlier.prompt | Enter a password you used before for this vault. You'll then set a new one. |
| earlier.note | Changing a password in TDrive doesn't stop earlier passwords from working, because they all protect the same key. Choose strong passwords and keep your recovery key safe. |
| device.reset.title | Set a new password with {deviceMethod} |
| password.new.body | This protects the same key. Your files aren't re-encrypted. |
| error.recovery.invalid | That recovery key doesn't match this vault. Check it and try again. |
| error.publish | Couldn't save the new password to Telegram. Your vault is unchanged. Check your connection and try again. |

6. **README replacement** (≤90 words, every claim checked against the set):
   > TDrive has no server and cannot reset a forgotten encryption password. Every vault has a recovery key: the master key itself, shown in Settings after you enter your password. Save it somewhere private; anyone holding it can decrypt your files, and changing your password does not change it. With the recovery key, an earlier password, or an enrolled Windows Hello/Android unlock, you can set a new password. Without any of these, encrypted files cannot be recovered. File and folder names are not encrypted.
