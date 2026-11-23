# Set E — Upstream comment for #140 + disclosure handling

TDrive master `52bc8de4`, 2026-11-23. Five rounds, six agents per round.
Note: the agents concluded the old-password property should be reported privately. Deepesh overrode that and mentions it publicly; see "Disclosure — RESOLVED" at the end.

## R1 — same goal: draft the first #140 comment (all six)

All six drafts converged on the same shape:
- Lead with the outcome.
- Correct Telegram/SMS kindly in 1–2 sentences.
- Propose a recovery key, which is the master key, shown after password re-entry.
- State the honest limit for already-locked-out users.
- Ask one yes/no question, offer a PR, and keep the AI disclosure.

**Unanimous (6/6): do NOT mention publicly that earlier passwords still work.** For the same reason, also drop "Try an earlier password" from the comment, because naming the feature reveals the property.
- The repo has no `SECURITY.md` (VERIFIED: `isSecurityPolicyEnabled: false`).
- `CONTRIBUTING.md` names no private channel.

## R2 — directed

- **How public the property already is (terra).**
  - README: *"Changing the password re-wraps the existing master key; it does not re-encrypt every stored file."*
  - `encryption.md`: *"rewraps the **same master key** … does not … rotate the underlying content key"* and *"cannot immediately revoke a device that remains offline"*.
  - Package doc: wraps are *"replicated into Telegram history"*.
  - **The implication is stated nowhere:** an old password + channel history = today's key. Verdict: **private only.**
- **Contact routes (astra).** No security inbox exists.

| Route | Notes |
|---|---|
| Issue tracker | Public |
| Telegram community `t.me/Tdrive_community` | A group, not private |
| X `@JeetRex` | DMs unconfirmed |
| GitHub Discussions | Disabled (`has_discussions: false`) |

- **Sentence-level fact check (fable).**
  - FALSE: *"no design can rescue an already-forgotten password"*. An enrolled device and the earlier-password route both exist. Scope the claim to "this feature".
  - MISLEADING: *"Forgot link in Encryption settings"*. It belongs on the unlock modal, since locked-out users can't reach settings.
  - MISLEADING: *"ChangePassword without the old password"*.
- **Tone (luna).** Jeet's register:
  - Replies are one to two sentences: *"Yeah, makes sense"*, *"Thanks for the suggestion!"*.
  - Standard capitalization, no emoji.
  - He answered #186 in **19m37s** and opened PR #199 **52m22s** after acknowledging #185.
- **Private note draft (sonnet-2).** Tightened in R3; see below.
- **Merged v2 (sonnet-1).** Paraphrase the no-reset comment instead of quoting it, so the comment doesn't read as calling him out. Leave out the orphan guard.

## R3 — directed

- **Biometric timeline (fable + astra, independently VERIFIED).**

| Date | Event |
|---|---|
| 2026-05-09 | Encryption first ships in v1.0.0 (*"There is no password reset if you forget your encryption password."*) |
| 2026-10-11 | v2.2.0 (no biometrics) |
| 2026-10-13 | PR #153 "Add biometric vault unlock" merged (branched off the v2.2.0 tag commit) |
| 2026-10-14 | di-kun reports the forgotten password |
| 2026-11-07 | **v2.3.0 = first release with biometrics** |

  - ⇒ The enrolled-device tip **cannot apply to di-kun** on any release build, since enrolling needs the password. Address it to "anyone reading later".
- **Red team (terra).**
  - "TDrive has no server, so nothing can verify you" is wrong: Telegram *can* verify the account. The real point is that verification can't produce the key.
  - Released macOS can't use Touch ID (ad-hoc signed), and there is no iOS release.
  - Logout deletes the enrollment.
  - The proposal *changes* the no-reset design, so the comment must not say it "keeps" it.
- **Luna's cut rejected.** It cited a fork (`ndgthuan/tdrive`), listed iOS instead of Linux, and dropped the AI disclosure.

## R4 — counter/falsify (v4, 202 words)

- **Data-loss risk (sonnet-1, fable).**
  - "Still unlocked" holds only while TDrive is open, because the key lives in memory until exit or lock. There is **no idle auto-lock**.
  - Closing the app ends an unlocked session.
  - "Or you set up Windows Hello" could be read as "enroll now", which needs the password.
  - ⇒ Say "already set up", and list close / sign out / disable / change password as don'ts.
- **Missed entry points (sonnet-1).** `preview-unlock-card.ts` is a separate inline unlock with no link to the modal, so it needs its own "Forgot password?". The CLI uses its own flow.
- **Platform reach (fable).** Android is the same Wails/Svelte frontend, but `AppRoot` picks a desktop or a mobile shell, so there are two entry points.
- **The private-note ask leaves #140 (luna, terra).** It is a second ask and draws attention to encryption (medium risk). Send it separately.
- **"Config message format stays the same"** invites scrutiny (terra, medium). Reword it.

## R5 — final sign-off (v5)

| Agent | Result | Note |
|---|---|---|
| sonnet-1 | **BLOCK** | "Nothing new is stored in Telegram" is false: a reset publishes one new config (`publishConfig`). Also "shared frontend" → "frontend + Go". |
| sonnet-2 | **BLOCK** | Same line. |
| fable | PASS | Optional: add "turn off device unlock" to the don'ts (Disable deletes the enrollment with no password). |
| luna | PASS | Tone/clarity. |
| terra | PASS | Disclosure. |
| astra | PASS | Private-contact wording, simplified. Venue ranking: X DM → one-line issue "Preferred security contact?" → Telegram group → never #140. |

All three fixes were applied in the final version below.

## ✅ FINAL — public comment for #140 (199 words)

> Coding isn't my forte, so I used Claude to read the encryption code and help draft this. Please correct anything I got wrong.
>
> **Proposal: a recovery key.** Encryption settings gets "Show recovery key": re-enter your password, then save the key somewhere safe. The unlock prompt (and the photo-preview unlock card) gets "Forgot password?": enter the key, set a new password. Under the hood that's the same as a normal password change: no new kind of data in Telegram, nothing re-uploaded or re-encrypted. It's shared frontend + Go, so desktop and Android both get it.
>
> On Telegram/SMS: Telegram can verify your account, but that can't produce the random key your files are encrypted with. Only a copy saved beforehand can. So this relaxes the current "no reset" design, but only for someone who saved their key. It can't help anyone already locked out without one.
>
> For anyone reading later: if TDrive is still open and unlocked on a device, or Windows Hello / Android unlock is already set up there, download what you need first. Until then, don't close the app, sign out, turn off device unlock or change the password there.
>
> @jeetrex17 would you accept a PR for this?

## ✅ FINAL — separate private-contact ask (NOT on #140)

> What private channel do you prefer for a TDrive security question?

Venue order: X DM `@JeetRex` → a new one-line issue "Preferred security contact?" → Telegram community group.

## ✅ FINAL — private note (send only once a private channel exists)

> Hi, Deepesh here (silicon-valley-mafia). I used AI assistance; facts checked at master 52bc8de4.
>
> **In plain terms:** changing a TDrive password doesn't cut off someone who knew an earlier password.
>
> **Why:** ChangePassword re-wraps the same master key under the new password and posts a new config message, but older config messages are never deleted from the channel. Anyone who can read the channel history and knows any earlier password can still unwrap today's key. Hints from every config are in that history in plaintext too.
>
> **Mitigation:** once a new config round-trips (fetch, unwrap with the new password, key check), delete the older config messages. That only helps going forward; copies already made stay valid. True revocation needs master-key rotation.
>
> **Separate bug:** if all config messages are gone, CreatePasswordContext mints a new master key even when encrypted files exist, orphaning them.
>
> I'm happy to send PRs for both. Would you rather discuss this publicly in #140 or keep it private until it's fixed?

## Disclosure — RESOLVED 2026-11-23

Deepesh decided to mention the property publicly in #140. Reading the channel needs the user's own Telegram login, because the app is local and uses the user's own API ID. The comment that was actually posted supersedes the "FINAL" drafts above, which omitted the property. The notes below are kept for the record.

### ✅ v6 — the comment to post (supersedes the 199-word version)

> Coding isn't my forte, so I used Claude to read the encryption code and help draft this. Please correct anything I got wrong. Longer notes: https://github.com/silicon-valley-mafia/public-shed/blob/main/jeetrex17/TDrive/Issues/140/exploration.md
>
> **On Telegram/SMS:** TDrive runs locally with your own API ID and Telegram login, and there's no server. Logging in proves you own the account, but it can't produce the random key your files are encrypted with. Only a copy saved beforehand can.
>
> **1. Recovery key.** Encryption settings gets "Show recovery key": re-enter your password, then save the key somewhere safe. The unlock prompt (and the photo-preview unlock card) gets "Forgot password?": enter the key, set a new password. Under the hood it's a normal password change: same master key, one new config message, nothing re-uploaded or re-encrypted. Shared frontend + Go, so desktop and Android both get it.
>
> **2. "Try an earlier password".** Changing the password re-wraps the same master key, and the older config messages stay in the channel, so any earlier password still opens the vault today. An explicit option on the unlock screen (followed by setting a new password) is the only code change I found that could help someone already locked out, if they changed their password at some point and remember an older one.
>
> The flip side: changing your password after it leaks doesn't lock out the old one for anyone who can read your channel, and reading the channel takes your Telegram login. Deleting older config messages after a successful change would fix that going forward without touching any files, but it would mostly rule out option 2. Your call on which matters more.
>
> **If you're locked out now:** if TDrive is still open and unlocked on a device, or Windows Hello / Android unlock is already set up there, download what you need first. Until then, don't close the app, sign out, turn off device unlock or change the password there.
>
> @jeetrex17 would you accept a PR for the recovery key first? The rest could follow as small separate PRs.

### Original notes

- `silicon-valley-mafia/public-shed` is **PUBLIC** (`visibility: PUBLIC`, 0 stars, 0 forks). Sets A–D under `jeetrex17/TDrive/Issues/140/sets/` already describe the old-wrap property, including cracking-rate figures (Set A, Set C).
- This contradicts the R2 verdict of **private only**.
- Options, for the owner to decide:
  - (a) Make `public-shed` private.
  - (b) Move `Issues/140/` to a private repo and delete it from `public-shed`. Note that git history keeps it unless rewritten.
  - (c) Accept the exposure. The building blocks are already in TDrive's own docs, but the implication is not.
