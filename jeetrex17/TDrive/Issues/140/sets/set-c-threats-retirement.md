# Set C — Threat model, old-wrap retirement, hints, password policy

TDrive master `52bc8de4`, 2026-11-23. Five rounds, six agents per round.

## R1 — same goal: red team (all six)

All six produced the same top five threats:

1. **Old wraps are kept forever.** Every password the user has ever set still opens today's key. Critical.
2. **The password minimum is weak.** `validateNewPassword` only checks `len(password) < 8`, and it counts **bytes**, not characters. High or critical.
3. **Hints are plaintext.** Every hint ever set is readable by anyone with channel access. Medium to high.
4. **A recovery key could leak** (proposed feature). It can't be brute-forced, but anyone who obtains it gets the master key permanently. Critical, but low-to-medium likelihood.
5. **Reset from biometrics could be abused** (proposed feature). A stolen device plus a coerced or known unlock could give an attacker lasting control. High or critical, but low likelihood.

Who can read a personal channel:
- The owner, any admins they appoint, and subscribers. *"New subscribers can see the entire message history"* (Channels FAQ).
- Anyone holding a hijacked logged-in session.
- Telegram itself. Cloud chats are not end-to-end encrypted; Telegram calls them "theoretically accessible" and says individual engineers can't access them.
- The personal drive is a broadcast channel (`CreateBroadcastChannel`).

Three agents (luna, terra, astra) said **never allow a biometric-only reset**. That was settled in R2/R5.

## R2 — directed

- **Retiring old messages (sonnet-1):**
  - Sync ignores remote deletions of control messages. `tombstoneDeleted` only maps live file messages, so **`replay_log` keeps old wraps on every existing device.**
  - The guard `validateHardDeleteBodyTargets` only allows file bodies, so retiring old configs needs a new delete path.
  - Rules: delete only after the new config is published, projected and confirmed. Never delete the newest. Treat the delete as best-effort and idempotent.
- **Master-key rotation (sonnet-2):**
  - There is no re-encrypt machinery (no `reencrypt`/`rewrap` code anywhere). `source_copy` copies bytes.
  - There is no key-id field on file ops.
  - A keyring would need config v2. **Out of scope for #140.**
- **Password policy (fable):**
  - Keeping 100 RTX 4090s running for 10 years below a 1% success chance needs **at least 52.25 bits**.
  - That means 12 random lowercase letters, 9 random alphanumerics, or **5 EFF words** (64.6 bits). Four EFF words is 51.7 bits, a 1.47% chance, and fails.
  - `nbutton23/zxcvbn-go` is MIT but has been archived since 1 Feb 2023. `@zxcvbn-ts/core` 4.2.0 is MIT and 10.9 kB gzipped.
  - HIBP's k-anonymity API still sends 20 bits of the hash plus your IP to a third party. An offline top-100k list is about 0.8 MiB.
- **Hints (luna):**
  - NIST SP 800-63B-4 §3.1.1.2 item 7: *"Verifiers and CSPs SHALL NOT permit the subscriber to store a hint (e.g., a reminder of how the password was created) that is accessible to an unauthenticated claimant."* Rev 3 §5.1.1.2 had similar wording.
  - Bitwarden: *"When requested, the hint is sent to the account holder's email."*
  - macOS shows the hint after 3 wrong attempts.
- **Reset vs unlock (terra):** reset gives an attacker a password they know and can lock out the owner. Unlock already exposes all the data. Not forbidden outright, but it is a higher-impact capability.
- **Deleting messages (astra):**
  - `channels.deleteMessages` removes the message for all subscribers, with no stated age limit.
  - Privacy Policy §10.2's 48-hour retention applies to **supergroups**.
  - `Gotd.DeleteMessages` has **no media guard**, so control-message ids can be deleted.
  - Deleting is hygiene, not revocation: it invalidates **zero** copies an attacker already made.

## R3 — directed

- **Retirement vs "try an older password" (sonnet-1):**
  - Retirement breaks "try older password" only on **fresh** devices. Existing devices keep `replay_log`.
  - If the newest config itself got deleted, a fresh device would see `exists=false`, and `CreatePassword` would mint a new key.
- **Orphan guard (sonnet-2):**
  - The `exists=false` path needs all of: empty replay, a successful refresh, `GetEncryptionConfig` returning not-found, and an empty second rebuild.
  - Proposed guard: refuse to create a new vault when encrypted file ops exist. Proposed issue title: *"Refuse to create a new vault password when encrypted files already exist without an encryption config."*
- **Residual risk (fable), at 170,000 guesses/s across 100 GPUs.** Example history: an old password of 8 random lowercase letters, a current one of 5 EFF words.
  - Attacker who copies the channel **after** retirement: about **2.65 million years** expected.
  - Attacker who copied it **before** retirement: about **7.1 days** expected, via the old wrap.
  - Telegram-side attacker: unknown retention, so between those two.
- **Account layer (luna):**
  - `session.json` holds the raw gotd session bytes, with 0600 file / 0700 folder permissions and **no encryption or keyring**.
  - Nudges worth adding: enable Telegram 2-Step Verification, review active sessions, encrypt the disk.
- **Pending-reset protocol (terra):**
  - Old clients skip unknown ops, so they would never show the warning.
  - There is no device identity, only the account id.
  - Even a 24-hour delay guarantees nothing.
  - Verdict: **prefer a recovery key set up beforehand.**
- **What "biometric" really means (astra):**
  - **Windows:** `RequestSignAsync` accepts the Hello **PIN or** a biometric. encryption.md: *"Windows Hello can offer its OS PIN."*
  - **Android:** `BIOMETRIC_STRONG` only, no device credential. Uses `setUserAuthenticationRequired(true)` and `setInvalidatedByBiometricEnrollment(true)`.
  - So someone who knows the PIN can unlock on Windows but not on Android.

## R4 — counter/falsify

- **Re-ranked (sonnet-1):** **#0 is theft of the local session** (`session.json`), which grants channel read access. The attacker then cracks the *current* wrap. Old-wrap retention is a strict subset of that path (it also needs an old weak password).
- **Orphan guard as written is unsafe (sonnet-2):**
  - On a fresh install `replay_log` is **empty**, so the guard fails open.
  - `LIKE '%|enc=1%'` is case-insensitive and also matches `enc=10`.
  - Instead, use a parse-based check that runs only after `EnsureAuthoritative` (requires `initial_sync_done=1` and `needs_projection_rebuild=0`).
  - Don't rely on `SearchChannelMessages` for this.
- **48-hour claim retracted (fable).** Privacy §10.2 says, in full: *"In supergroups and channels, deleting a message removes it for all participants. Note that deleted messages and original versions of edited messages from supergroups are stored for 48 hours after deletion…"* Channels do have an admin log, but no documented retention for deleted broadcast-channel messages. **Treat server retention as unknown.**
- **NIST and hints (luna):** the 800-63B-4 wording is confirmed (above). Bitwarden's email-the-hint behaviour and macOS's 3-attempt rule are both confirmed.
- **Windows PIN (terra):** confirmed. There is no `UserConsentVerifier` and no biometric-only enforcement. Android: `minSdk 21`, biometrics from API 23, and the invalidation setter only from API 24.
- **Frontend checks (astra):** the UI rejects `.length < 8` (UTF-16 units), so two emoji pass the backend but fail the UI. **There is no strength meter.**

## R5 — counter/falsify (final)

- **Retirement rule (sonnet-1):**
  - Remote deletion is harmless to other devices.
  - `planHistory` tolerates gaps ("thinned by deletions").
  - **One invalid old config fails the whole rebuild closed**, so pruning also removes that hazard.
  - **"Keep 3 / wait 30 days" was rejected**: every old wrap is an equally old copy of the same key, so age doesn't matter.
  - **Final rule:** after the new config round-trips (fetch → unwrap with the new password → key check passes), delete **all** older configs, choosing only ids whose local `op_type` is `encfg`.
- **Rotation stays out of scope (sonnet-2).** The mount and media readers reject `EncryptionVersion != 1`, and `DecryptStream` rejects a non-zero flags byte. Minimum work is about 400–700 lines plus a config v2 format. It does nothing for a forgotten password. Follow-up issue title: *"Encryption: versioned keyring and master-key rotation (design)"*.
- **Arithmetic re-checked (fable).** All figures are correct. Correction: "3 EFF words, 7.4–29.8 y" were *exhaust* times. At 1,700–1,800/s on one GPU, the expected time is **about 4.1–4.4 years**.
- **Case against removing hints (luna):**
  - Alomar et al. (30 participants): cue-based recall cut authentication failures by 30%.
  - Adobe: about 7,000 hint→password matches among 109,305,888 hint-bearing records.
  - **Balanced recommendation:** stop publishing hints to Telegram. Keep an optional local, OS-gated hint, warn when the hint contains part of the password, and ship the recovery key before removing hints.
- **Reset via Windows PIN (terra).** A PIN holder can already unlock and download everything. A reset adds owner lock-out and an attacker-known password. Prefer a recovery key that is independently protected; any reset should be opt-in.
- **Session theft (astra).** A gotd session is portable to another machine (the auth key is tied to the user). It appears in Active Sessions, so it can be revoked. Telegram Desktop encrypts `tdata`. **TDrive's plaintext `session.json` is below that standard**, and encrypting it is a worthwhile companion fix, not a form of password recovery.

## SET C — conclusions

1. **Threat order:** local session theft (#0) → cracking the current password (weak 8-byte minimum) → old-wrap retention → plaintext hints → recovery-key leak / reset abuse (proposed).
2. **Retirement:** after the new config is verified by round trip, delete **all** older `encfg` messages. Use a new delete path filtered on `op_type=encfg`. It is hygiene for future channel readers only; copies already made remain dangerous.
3. **Password policy:** generated 5-word passphrase by default; for typed passwords, at least 12 code points plus zxcvbn-ts; offline blocklist; HIBP only if opted in.
4. **Hints:** stop publishing them to the channel. Keep an optional local hint until a recovery key exists.
5. **Biometric reset:** **not** biometric-only on Windows, because the Hello PIN is accepted. Android `BIOMETRIC_STRONG` is defensible, but prefer the recovery key.
6. **Separate issues to file:**
   - (a) Orphan-key guard in `CreatePasswordContext`, parse-based and only after an authoritative sync.
   - (b) Encrypt `session.json`.
   - (c) Keyring/rotation design, later.
