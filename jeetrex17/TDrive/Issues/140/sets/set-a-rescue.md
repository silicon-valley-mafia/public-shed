# Set A — Round 0 refresh + rescuing a user who already forgot (di-kun)

TDrive master `52bc8de4`, 2026-11-23. Six agents ran every round: claudesonnet-1, claudesonnet-2, claudefable, gptluna, gptterra, gptAstra.
**VERIFIED** = an agent read it in source. **REASONED** = inferred. **MEASURED** = benchmarked in a sandbox.

## Round 0 — refresh of known facts

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 0.1 | KEK = Argon2id 64 MiB / t=3 / p=4, 32-byte key, 16-byte salt | VERIFIED | `backend/crypto/vault.go` `DefaultParams`, `DeriveKEK`; params stored as JSON, bounded by `normalizeParams` |
| 0.2 | Master key = 32 random bytes, wrapped XChaCha20-Poly1305 into a 72-byte blob (24+32+16), nil AAD | VERIFIED | `vault.go` `NewMasterKey`, `WrapMasterKey`; any AEAD failure surfaces as `ErrWrongPassword` |
| 0.3 | Key-check `tdrive-key-check-v1` is sealed under the MASTER key (59-byte envelope), nil AAD | VERIFIED | `vault.go` `EncodeKeyCheck`, `VerifyKeyCheck` |
| 0.4 | File subkeys = HKDF-SHA256(master, per-file 16-byte salt from the TDE1 header, info `tdrive/file/v1`) | VERIFIED | `backend/crypto/stream.go` `deriveSubkey` |
| 0.5 | Crypto location | CORRECTION | Lives in `backend/crypto/`, not `tdcrypto` |
| 0.6 | `ChangePassword` re-wraps the SAME master key with a fresh salt; only `CreatePasswordContext` calls `NewMasterKey` | VERIFIED | `encryption/service.go` |
| 0.7 | `CreatePasswordContext` returns `ErrPasswordAlreadySet` | VERIFIED + CORRECTION | `mountpolicy.EnsurePersonalConfig` tries local replay first and refreshes from the channel only when none is found; a failed refresh fails closed with `ErrEncryptionPolicyUnavailable` |
| 0.8 | Config is published to the channel before it is stored locally | VERIFIED | `publishConfig` → `emitOp` → `EmitAndProject` (send, then `ProjectFromOp`) |
| 0.9 | No code deletes old `encfg` messages | VERIFIED | Confirmed again in R4/R5: `validateHardDeleteBodyTargets` only admits `file_upload`/`file_part` ids |
| 0.10 | A biometric enrollment holds the real master key, and `Provider.Unlock` returns it without the password | VERIFIED | `encryption/biometric.go` `BiometricEnrollmentKey`; `biometric/biometric.go` `EncodeSecret` (32-byte scope digest + 32-byte key) |
| 0.11 | `VerifyBiometricKey` returns only an error: it checks the key-check, then calls `StoreMasterKey` | CORRECTION | |
| 0.12 | No biometric provider on Linux | VERIFIED | `provider_unsupported.go` → `ErrUnavailable` |
| 0.13 | `ChangeEncryptionPassword` deletes the biometric enrollment BEFORE calling `service.ChangePassword` | VERIFIED | `internal/app/encryption.go`: "Native enrollment is removed first." |
| 0.14 | Where enrollments are stored | VERIFIED | Windows: `<datadir>/biometric/<scope>.windows-hello.json`, AES-GCM under an HKDF of a Windows Hello signature (not DPAPI). Apple: Keychain, `WhenUnlockedThisDeviceOnly` + `BiometryCurrentSet`. Android: `noBackupFilesDir/biometric-vault/` under an AndroidKeyStore AES-256 key |
| 0.15 | Hint is plaintext (URL-escaped `hint=`) inside every published `encfg` op | VERIFIED | `buildConfigWithKDF`, `configOp`, `projection/wire.go` `Format` |
| 0.16 | No maximum hint length and no hint≠password check | VERIFIED (NOT FOUND in code) | |
| 0.17 | `readEncryptionReplay` applies every config in ascending order; the local `encryption` table is keyed on `channel_id` | VERIFIED | `projection/encryption_rebuild.go`, `apply.go`, `schema.go` |
| 0.18 | Unknown op types fail with `ErrWireBadOpType` and are skipped before projection ⇒ a new op type needs a history rescan | VERIFIED / REASONED | `projection/wire.go` `Parse`, `sync/parse.go` |
| 0.19 | Header tampering is detected and fails closed | VERIFIED, conditional | Only for a parseable change that gets re-observed; incremental sync ignores ids at or below the watermark |
| 0.20 | No encryption, biometric or vault-UI changes in 26e7f1a3..52bc8de4 | VERIFIED | |
| 0.21 | "Remembered" = master key held in RAM only; no keyring, DB or file copy | VERIFIED | `StatusContext`, `RememberPassword`, `StoreMasterKey`; the daemon persists no key |

## R1 — same goal for all six: what can rescue di-kun today?

All six converged on this ranking:

1. **A still-running unlocked session.** Export everything. It cannot reset the password, because `ChangePassword` needs the current one.
2. **A biometric enrollment that is still valid.** Unlock, then export.
3. **Hints.** The current one and every old one are readable in channel history.
4. **A remembered OLD password.** It unwraps the same master key through its old `encfg`, but `UsePassword` only reads the current config, so the app can't use it.
5. **Offline guessing of the user's own candidates.**
6. **Telegram/SMS:** no route exists.

## R2 — directed

- **Biometric scope (sonnet-1):**
  - `biometricConfigHash` = SHA-256 over domain, AccountID, ChannelID, Version, KDF params, KDFSalt, WrappedMasterKey and KeyCheck. CreatedAt and Hint are excluded.
  - A password change on device A leaves device B's enrollment working **until B syncs**. After that, `ErrInvalidated`.
  - That refusal is **policy, not crypto**: the master key never changed.
- **Historical-wrap unlock (sonnet-2):**
  - `replay_log` keeps `raw_header`, so old configs are available locally.
  - Design: `ListHistoricalEncryptionConfigs` (newest first, skip tampered rows, parse and validate) → `UseHistoricalPassword` (unwrap, then `VerifyKeyCheck` against the CURRENT config) → `StoreMasterKey` and force a new password via `SetPasswordWithUnlockedKey`.
  - Cap how many rows are tried and how much memory each row may use.
- **MEASURED (fable):** Argon2id 64MiB/t3/p4 takes **152.4 ms per guess on one Xeon 8370C vCPU** (Python `cryptography`).
- **Telegram/SMS (luna, web):**
  - `auth.sendCode`, SRP and `recoverPassword` only authorize a session.
  - The Gateway API only delivers verification codes.
  - Mini App CloudStorage holds 1,024 keys × 4,096 chars, readable by that bot's Mini App. SecureStorage is device-local and holds 10 items.
  - Saved Messages is cloud storage, not end-to-end encrypted.
  - SMS would need a server, and TDrive has none.
  - Plain-English answer for Jeet: *a Telegram login or phone proves you control Telegram, not that you know the TDrive password, and it cannot rebuild a random key. Recovery needs a second copy of the key escrowed beforehand, and whoever holds that copy joins the trust boundary.*
- **Reset from a session (terra):**
  - Reuse the `ChangePassword` sequence with the key supplied, checked against the existing key-check. Never call `NewMasterKey`.
  - A reset must **publish first, then delete** the enrollment (the opposite of today's order).
  - Prefer a fresh `Provider.Unlock` prompt over `LoadedMasterKey`. There is no standalone re-authentication primitive.
- **Partial salvage (astra):**
  - Photo-backup originals are left in place.
  - `tdrive.db` holds plaintext names, sizes and the folder tree.
  - Gallery and image caches of encrypted files are ciphertext.

## R3 — directed

- **Metadata (sonnet-1):** encrypted files keep a plaintext `n=` (name), `sz=`, `psz=` (plaintext size) and `ts=` in the caption, plus `TDrive: <filename>`. The Telegram document filename and folder names are also plaintext. The README says: "File names, folder names, sizes, Telegram relationships, and operational metadata remain visible."
- **Offline helper (sonnet-2):** the response was blocked by the agent's content filter. The topic was redone in R5 as a data-format question.
- **GPU (fable):**
  - hashcat mode 34000 **cannot attack TDrive's verifier directly**. An attacker would need a custom kernel running Argon2id and then a Poly1305 check; the AEAD cost is negligible.
  - RTX 4090 at m=64MiB, t=3, p=1: 1,699 H/s (hashcat gist) and 1,667 H/s (security.SE 285414).
- **Prior art (luna):** Cryptomator: "We cannot reset the password of a vault for you in any way." Bitwarden: "no way to access, retrieve, or reset your master password" (offers a hint and copy-out from a logged-in session). 1Password: "no one can reset your password"; try previous passwords; Emergency Kit. Proton: active-session reset; 12-word phrase. Standard Notes: "password resets are simply not possible"; export while signed in. MEGA: start over; parked data kept **60 days**. Tresorit: active-session reset.
- **Enrollment survival (terra):** bind enrollment to account + channel + a stable key identity + an explicit enrollment-generation counter, not to salt/wrap. Today's invalidation can't revoke a copied key, an unlocked session or an offline device anyway. Verdict: keep enrollments across password changes, and offer an explicit, honestly worded revoke.
- **Multi-device rescue (astra):**
  - Biometric unlock on the enrolled phone or PC → select → Download. Max 500 selected rows per batch; folder trees up to 10,000 items; 32 parts per file.
  - There is no vault-wide "download all".
  - 50 GB at 10 MB/s ≈ 1 h 23 min 20 s.

## R4 — counter/falsify

- **"Old encfg never deleted" — SURVIVES.** `DELETE FROM replay_log` appears only in `DeleteChannel` (leaving a shared drive). No compaction.
- **"Biometric rejects after a synced change" — SURVIVES.** The providers call `DecodeSecret` with the CURRENT scope. A stale native item is not cleaned up on the Use path.
- **"An unlocked session can't change the password" — SURVIVES.** There is no CLI or daemon re-wrap path.
- **"The channel is public" — wording FALSIFIED.** vault.go: *"We treat the Telegram channel and any local Telegram cache as untrusted public storage for confidentiality"*, meaning attacker-readable, not safe to publish. A historical-wrap unlock adds no new exposure, but it adds no guess limit either.
- **`normalizeParams` — SURVIVES.** Memory 16–256 MiB, time 1–10, parallelism 1–16. `kdfMu` serializes derivations.
- **Numbers corrected (fable):**
  - "4 guesses/s" is the 250 ms *design target* in a comment, not a measurement.
  - The GPU is VRAM-bound (352 × 64 MiB ≈ 22.5 GB), and throughput scales with 1/(m·t), not with p.
  - **About 1,700–1,800 guesses/s per RTX 4090.**
  - Specops' "490 H/s on 8×5090" gives no parameters and is **unusable**. The earlier "61 H/s per card" proxy is withdrawn.
- **Telegram key material — narrowly falsified.**
  - The MTProto `auth_key` is stable per session, so a key wrapped under it beforehand could be recovered while that session survives.
  - Passport's `passport_secret` is protected by Telegram 2FA.
  - Neither helps without setup done beforehand.
  - MEGA support can revert to a remembered old password, and 1Password organizers can recover members, but both need something set up in advance.
- **Plaintext on disk (terra):**
  - Download staging (`.tdrive-download-*`) is plaintext.
  - Android copies completed downloads to public Downloads.
  - WebDAV serves decrypted bytes, which the OS may cache.
  - The DB is plain modernc SQLite.
  - There is no option to encrypt file names.
- **Shipped biometrics (astra):** **only Windows AMD64 and Android ARM64.** The macOS ARM64 release is ad-hoc signed, so Touch ID fails. iOS exists only in source.

## R5 — counter/falsify (final)

- **macOS — VERIFIED.**
  - `package-mpv-darwin.sh` runs `codesign --force --sign -`. There is no entitlements file and no notarization.
  - `provider_apple.m` always uses the data-protection keychain, so it reports `signing_required`.
  - Self-built binaries signed through `build/darwin/Taskfile.yml` `codesign:bundle` can work.
- **Telegram export (sonnet-2):**
  - JSON/HTML exports include message `text`.
  - Caveat: *"public group and channel exports will only contain messages sent by the user requesting the export"*.
  - Wire format: `TDX1|t=encfg|salt=|kdf=|wrap=|check=|hint=|cv=`, with base64 RawURL fields.
  - Exported Go functions: `DeriveKEK`, `UnwrapMasterKey`, `VerifyKeyCheck`, `UnmarshalParams`, `ValidateVaultMaterial`.
- **Laptop rate corrected (fable):** the Argon2 spec's Table 4 gives a 2.35× speedup at 4 threads. **Laptop ≈ 9–33 guesses/s (~13/s typical).** The memory-bandwidth ceiling is about 124/s at 50 GB/s, but CPU cores run out first.
- **Passport escrow — NOT VIABLE.** `secure_settings` is a typed Passport setting, and overwriting it breaks the user's real Passport.
- **"Can only start over on a new channel" — FALSIFIED (terra).** `personaldrive.Select` does not exclude channels that have a vault, so a user can re-select the old channel; unlocking still needs the password. Limits re-verified: a selection over 500 rows is rejected whole; 10,000 items per tree, including the root. There is no disable-encryption or decrypt-in-place feature.
- **Plaintext leaks (astra):** download staging is removed on error, but a crash can leave partial plaintext (no startup sweep). WebDAV returns 403 while the vault is locked. **CONTESTED:** whether video scrub frames are written to disk for *encrypted* videos.

## SET A — conclusions

1. **Nothing in TDrive today can rescue a user with no unlocked session and no valid biometric enrollment.** Telegram/SMS can't, by construction.
2. **Rescue order today:**
   1. A still-unlocked session → export.
   2. A Windows or Android enrollment that hasn't seen a password change → export.
   3. A remembered OLD password (needs new code).
   4. Hints.
   5. Guessing the user's own candidates at about 13/s on a laptop, or about 1,700/s on a GPU.
3. **Cheapest code that would have helped:**
   - (a) "Try an older password" against historical `encfg` rows.
   - (b) Re-wrap from a freshly biometric-unlocked session, publishing before deleting the enrollment.
   - (c) Keep enrollments across password changes.
4. Names, sizes and the folder tree are plaintext anyway. A locked-out user keeps the inventory, not the contents.
