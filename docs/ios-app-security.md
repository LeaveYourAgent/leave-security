# Leave iOS app: security design

**Status as of 2026-10-30.** The app has been on the App Store since 2026-10-19
(build 68, version 1.0); the current store build is 81 (version 1.5.0, on sale
2026-10-27). Android has been on Google Play since 2026-10-24 (1.2.0, code 12);
1.3.0 (version code 18) since 2026-10-25, 1.3.1 (code 19) the same day, 1.4.0 (code 20)
on 2026-10-26, and the store's current Android version is 1.5.0 (code 21, on the
listing 2026-10-27). From code 19 the Android release build has code shrinking (R8) on,
so libraries are shrunk and renamed while Leave's own classes keep their names by design;
the sign-in code field is reachable by screen readers, with no change to how the code is
handled. 1.4.0 (build 80, code 20) adds Talk's web search on the phone's side, described
in the Talk row. 1.5.0 (build 81, code 21) adds state rules on a deal, on since 2026-10-30
and described in their own row. The crypto files are unchanged across all of these:
leave-crypto tags `ios-78`, `ios-80`, `ios-81`, `android-19`, `android-20` and
`android-21` are the same commit, and both pipelines' drift checks matched. The app code
is the same on both platforms, so a build 78 fact holds for every later build unless a
line says otherwise. Each item is marked **Live** (in the App Store build today), **In testing**
(in a TestFlight or Play internal build only), or **Planned** (designed, agreed with
the backend, not yet in code). A "Live (build 72)" mark means the fact has not changed
since that build; "build 78" marks what changed in 1.3.

Checked 2026-10-25 against the build 78 source (the commit tagged `ios-78` in
github.com/LeaveYourAgent/leave-crypto matches the app's crypto files).

This is the public edition of the app's internal plan. It is the app-side counterpart
to the Leave backend plan and the Leave API contract, both published in this
repository. Route shapes and cryptographic framing are defined in the API contract;
this document says what the app does with them.

Principle: wherever a choice trades convenience or cost against the athlete, the
athlete wins. Keys never leave the phone. Nothing private is readable by Leave, by
our hosting provider, or by a vendor outside a session the athlete started.

## Status summary

| Area | Status |
|---|---|
| Sign-in with an emailed six-digit code, no password anywhere | Live (build 72; since build 13 the code is sent and verified by the API; the app registers this phone's Secure Enclave key at sign-in) |
| Sign-up: who the account is for, names, athlete's date of birth as an age gate (13 and up; 13 to 17 through a parent or legal guardian), clickwrap agreement with the accepted terms version recorded | Live (build 72) |
| Terms of Service and Privacy Policy links on every sign-in and sign-up screen and in Profile | Live (build 72) |
| Delete my account, in-app, with typed confirmation | Live (build 72): deletes the account on the server and wipes the phone. A deletion is also possible from the screen shown when the server no longer accepts an old build. For a parent or guardian that needs the linked athlete's account to be named, so from build 61 the guardian's phone keeps the linked athlete's account id, first name and account state in its encrypted store, replaced whenever the server answers and removed at sign-out and at deletion. In builds 43 to 60 a guardian on a refused build could not delete either account from that screen; walking it by hand found it) |
| Deleting without the key: a phone that does not hold the key can only ask, with a code emailed to the account; nothing is deleted for 7 days; every approved phone, the account's email and a 13 to 17 athlete's guardian are told; an approved phone, or any use of Leave on one, cancels it; an account with no approved phone deletes with the emailed code alone | Live (build 72; from build 39) |
| Local storage encrypted with AES-256 under a 192-bit random key (24 bytes from the platform random generator, written as 32 base64url characters because the store library takes its key as text) held only in this device's Keychain (never synced); the store is excluded from backups; the app refuses to run unencrypted. Corrected in build 78: builds 77 and before opened the store with the library's default, AES-128, which reads only the first 16 bytes of a key written as 32 hexadecimal characters, so 64 random bits protected it. Build 78 moves the stores to a new file under the new key on first launch, deletes the old files and removes the old key; the move runs again if interrupted. Android code 18 does the same, with the key sealed under the phone's hardware-backed Keystore key as before. The store library's encryption is AES-CFB with no integrity check at any key length: it keeps the store unreadable, not tamper-evident; the store was never in a backup and never left the phone. One exception: the crash-report queue is a second store, not encrypted, holding only code locations as numbers and build numbers. From build 40 a sign-out or a deletion erases the store's file, not only its index. What stays on the phone after a sign-out (from build 64): that account's key share in the Keychain, the phone's device key for it, the fact that the address has an account (the address alone, no name or role), and the names of files this phone uploaded, which exist nowhere else; each is read only for that account. The account's part of the phone's settings (which threads were opened, what has already been shown) is cleared at sign-out, so a second person signing in is shown the AI consent sheet and the recovery code's mention as on a new phone. Accessibility choices and the app lock are the phone's own and are kept at sign-out and at deletion. A deletion removes everything else | Live (build 72) |
| Data minimization: the phone keeps only the birth year and an under-18 flag; the server keeps an athlete's date of birth to apply age rules and no date for a guardian or Teammate (corrected 2026-10-01: this row said the birth date was not stored); sign-out drops names; delete wipes the store | Live (build 72) |
| App-switcher snapshot covered, temporary picker and camera files cleared | Live (build 72) |
| Release builds enforce strict App Transport Security (no local-network exception) | Live (build 72) |
| Certificate pinning: the app accepts Leave's API and MCP hosts only over certificates that chain to Google Trust Services roots (four) or ISRG (Let's Encrypt) roots (four: X1, X2, Root YE, Root YR), so a move off Google's certificates would not lock out installed copies; the hosts' CAA records still allow only Google. Rotation: the pinned roots are two certificate-authority families; a change would ship in an app update before the certificates move | Live (build 72; since build 45; Google's roots since build 21) |
| Privacy manifest declares what the app sends (email, name, date of birth, contracts and files, account and device identifiers, usage), all linked to the account, none used for tracking | Live |
| Dependabot alerts and continuous checks on every change: typecheck, lint, tests, `npm audit`, registry signature and provenance verification. From build 78: OSV-Scanner over the npm, Android (Gradle), Swift package and CocoaPods lockfiles; package versions a change adds must be at least 7 days old or accepted by name; Semgrep with Leave's own rules (blocking) and the TypeScript, React and secrets registry rules; every report PDF the app can write is checked against PDF/UA-1 with veraPDF; the published crypto copy is diffed against the app's files in both release pipelines; CI actions pinned by commit | Live (build 78) |
| No analytics SDK, no advertising SDK, no Google Analytics in the app | Live |
| System permission prompts only for the camera, photo library, microphone, speech recognition, Face ID and notifications, each with a plain-language reason | Live |
| No location, contacts or advertising-identifier access | Live |
| Export compliance: the app's encryption uses standard algorithms (AES-GCM, HKDF, P-256, HPKE, Argon2id) and is sold in the United States only; Leave answers App Store Connect's encryption question by API for each build on that basis | Live |
| Athlete-held encryption (the design in section 2) | Live (build 72) for contracts and files (from build 15): the share, every data key, the share wrap, the account key and the session grant run in the native module; the athlete's file is encrypted on the phone before upload and decrypted on the phone to open; the explained terms come back as one encrypted blob the phone opens |
| Face ID lock, screenshot protection on private screens, clipboard expiry | Live (build 72; from build 21): the app asks for Face ID or the passcode at launch and after a minute away, behind a cover that is up before the app draws anything; private screens come out black in screenshots and recordings; private text copied from a contract leaves the clipboard after 30 seconds, and a copied Talk reply after 2 minutes (build 78), and neither is offered to other devices. From build 78 (Android code 18) every screen is classified private or public, with a test that no screen is left out: private now includes saved conversations, reports, export, upload, the Teammate and Devices screens, the NIL payment screens (your rate, get paid, report a deal, your rules, what brands see), and the sign-in, sign-up and deletion screens; public is the fixed-text screens (plan prices, the consent sheet's lists, lessons, settings). Also from build 78, on iPhone, a third-party keyboard is refused while a secret field is up (the emailed sign-in code, the recovery code); everywhere else, Talk included, other keyboards stay, so people who rely on an accessibility or another-language keyboard keep them (decided 2026-10-24). On Android there is no keyboard block; the native recovery-code field asks keyboards not to learn from it (no personalized learning) Limit: iOS has no supported way for an app to block screenshots. Leave uses the system's secure text field to keep private screens out of them, which Apple does not document for this purpose, so an iOS update could stop it working; it is checked on a real phone at each release. Build 47 fixed a crash when the app was closed after signing out, caused by how that field was attached to the window. From build 57, internal tester builds run with the screenshot protection switched off, except on the recovery code screen, so that faults can be reported with screenshots; the choice is made when the app is compiled, a build for the App Store contains no code that skips the protection, and the archive script reads the built app and refuses one that does not match. The lock cover, the app-switcher cover and the clipboard expiry are on in every build |
| New-device approval and recovery code | Live (build 72; from build 22): a new phone gets the athlete's key from iCloud Keychain or from a phone the athlete approves in Profile → Devices; the approval wraps the key to the new phone's own Secure Enclave key-agreement key and is signed by the approving phone; until it arrives the new phone shows only the waiting screen. The recovery code is live, from build 48: 28 characters of Crockford base32, made in Profile behind Face ID or the passcode, shown once in a native view and never passed to JavaScript or stored; the key share is sealed under an Argon2id key (64 MiB, 3 passes) derived from it and only that sealed blob is kept on the server. Deleting one private strand is also live, from build 48. From build 52 the code can be copied or printed from that native view, still without passing through JavaScript: Copy puts it on the phone's pasteboard as a local-only item (not offered to Universal Clipboard) with a 5-minute expiry; Print opens Apple's print sheet with one page holding the code and the date and no name or address. Limits: while it is on the pasteboard, an app the person pastes into receives it; Apple's print sheet is the system's screen drawn over Leave's, so the screenshot protection does not cover it (a screenshot taken while the print preview is up shows the page, and the sheet's own share button can save the page as a PDF); a printed page is wherever the person puts it. The code screen itself still comes out black |
| A code to compare when approving a phone | Live (build 72; from build 45): both phones show six digits worked out from the new phone's own key and the request, for the athlete to compare before approving. Limit: the code stops approving the wrong phone and an intercepted sign-in code; it does not stop a Leave server that swaps the key, because six digits with no commitment step can be matched by trying keys. Closing that is planned after the first release |
| One key slot per Leave account | Live (build 72; from build 45): the key share is one Keychain item per Leave account, named by the account id; an arriving share is kept only if it is that account's key. A deletion removes the account's share from the Keychain; a sign-out keeps it, so signing in again needs no other phone (build 51; builds 45 to 50 left the share in place after a deletion) |
| A new install | Live (build 72; from build 45): a new install ends any session an earlier install left in the Keychain before anything runs, and removes its push registration; the athlete's key share is kept |
| Profile → Your key (status, rotate, revoke) | Planned, after the first release. Delete account does not run Revoke today; it deletes the records and keys on the server |
| On-device contract text extraction and OCR before upload | Live (build 72; from build 15): PDF text layer, Vision OCR for scans and photos, a Word reader; Google's OCR only after the athlete agrees on screen |
| On-device speech recognition only | Live (build 72; from build 25): the phone turns speech into text itself and sends the text; every request requires on-device recognition, so audio never leaves the phone. Where the phone has no on-device model for the language, voice says it is unavailable and typing carries on; there is no server fallback |
| Connections: public-only AI connections, per-client private unlock with retention notice | Live (build 72; from build 21): the list of connected AI apps, Share private data behind that app's retention notice and a confirmation signed by the phone, unlock for 15 minutes or 1 hour with a countdown, Lock now, Disconnect; never shown for 13 to 17 Each unlock of private data for an AI app asks for Face ID or the passcode (build 45) |
| Talk to Leave: every turn streams from the API with a grant for the strands it may read; what Leave saves comes back as a receipt the phone opens; since app 1.2 (build 72) the conversation and up to 200 saved conversations are kept on the phone in the encrypted local store, out of backups, and deleted at sign-out; Leave's servers keep none of it. From 1.3 (build 78) a reply that needs one contract's full terms, or why one Web thread fits, asks the phone for that one item's key in the middle of the turn, and the phone answers with a grant for that single record, so a Talk turn starts with no contract or thread keys at all. From 1.4.0 (build 80, Android code 20), on since 2026-10-26, Talk can search the public web for a question about something outside Leave: the server's AI writes one short topic from the question, checked to carry no name, school, team or place, and searches it with the market and sport from the public record; never the question itself, a strand, a contract term, a thread or the summary. The phone shows the answer only to you and keeps it only in the conversation on the phone, with each source's title and domain (never its address, which lives only in memory while the screen is open, and opens only when `https:`); in later turns the searched reply goes back to the server only as "[a web search answer]", never its words, titles, domains or addresses; and the two-year removal takes the sources with the text. Also from 1.4.0, a reply that is not whole after 120 seconds ends with a quiet line. A thumbs up or down on a reply or a report is kept on the server for 90 days, linked to the reply, with no words (1.3). The app can write a report as a tagged PDF (PDF/UA-1) on the phone; the server's report kinds were switched off for part of 2026-10-25 while a fault was fixed and are on again since 2026-10-25 | Live (build 78) |
| State rules on a deal, the disclosure window, and a deal's packet | Live (build 81, Android code 21; on since 2026-10-30): the athlete answers where they play (state, level and school) as a question; there is no location permission. For an athlete aged 13 to 17 the parent or guardian sets it, and the server records who set it and when, with no values. The rules are public data: a copy ships in the app and is refreshed from a route that takes no sign-in and carries only the state, level and school. Each deal shows that place's rules and, where a rule's window is counted from the signing, the day it ends ("The disclosure window in {state} ends {date}"). The signing date comes from the contract's reading, or from the athlete if they type it; the signing date, the day a disclosure window ends and the day the athlete marks a deal reported are kept on Leave's servers unencrypted, beside the payment dates, so a reminder can go the day before and the morning of. That notification carries one fixed sentence, with no brand, state or amount. A deal's packet is written on the phone from the contract it has opened and the rules, and the athlete shares it as a PDF or as text through the share sheet; Leave sends it to no one, and the PDF file is removed when the sheet closes |
| Leave Web live: strands, threads and signals read from the API, thread text opened on the phone, a run of the matcher with a grant that holds only the strand keys. From API contract v1.13.0 a run is accepted at once and done by a queued task: the sealed grant travels in the task, is opened only by the instance doing the work, and expires 15 minutes after the phone made it, the same as a contract's reading. The app follows the run's state from build 55 | Live (build 72; from build 25; the queued run from build 55) |
| Export my data: the server packs every private record as the ciphertext it stores, without reading any; the phone downloads the archive, opens each record with the key it holds and offers a plain zip through the share sheet, deleting its own copies afterwards; asking is signed by the phone behind Face ID | Live (build 72; from build 25) |
| Teammate view: a Teammate or guardian reads, read-only, only what the athlete's phone has wrapped to their account key (contracts, Web threads) plus the public record; strands are never wrapped to anyone | Live (build 72; from build 27) |
| Usage meter, allowance nudges, purchases | Live (build 72; from build 20): plan status, usage against each allowance, credits and auto-refill; the plan and credit packs are bought through Apple in-app purchase (StoreKit 2) and granted by the server from Apple's signed transaction, with the website's Stripe checkout offered beside it on the United States App Store storefront (section 3, item 11; this row said "web only" until 2026-10-25); nudges arrive with push |
| Crypto module checked against the shared test vectors | Live (the app's XCTest target runs every vector kind and four samples generated by the service; the native files are published, as a copy of the app's source, at github.com/LeaveYourAgent/leave-crypto (Apache-2.0) with the shared test vectors; the repository is tagged per store build (`ios-72`, `ios-77`, `ios-78`, `android-17`, `android-18`) and both release pipelines refuse an archive whose crypto files differ from the tagged copy (build 78)) |

## 0. Architecture the app adopts

- **Two layers.** A native Swift package, `LeaveCrypto`, does everything that touches
  keys: the Secure Enclave key pair, the athlete share in the Keychain, per-record data
  keys, wrap and unwrap, session key exchange, and recovery-code derivation. It has no
  dependency on the app, on React Native, or on the sign-in provider, so it can be
  published and reviewed on its own. A thin bridge exposes a small typed API to the
  app's JavaScript. JavaScript never sees a share, a data key, or a recovery code; it
  sees record IDs, opaque ciphertext, and booleans.
- **Repositories are the seam.** Screens talk only to repository interfaces. Today the
  implementations are mocks; the production ones call the Leave API with a sign-in
  token, an App Check token, and, when private data is involved, a session grant.
- **A `SessionGrant` object** owns the 15-minute window: the server's public key, the
  set of record IDs whose keys were handed over, the expiry, and a `renew()` that
  re-hands the same keys. Every private-data call goes through it.

## 1. Identity: emailed code, designed screens kept (Live since build 68)

Sign-in is an emailed six-digit code through WorkOS Magic Auth. The app keeps its own
Email and Code screens and calls two Leave routes instead of a hosted sign-in page:

| App step | Route | Notes |
|---|---|---|
| Email screen, "Send code" | `POST /auth/code { email }` | always answers 202 so it cannot be used to discover accounts; rate limited |
| Code screen, "Continue" | `POST /auth/verify { email, code, device }` | returns access and refresh tokens, whether the email is new, and whether this device is approved |
| Refresh | `POST /auth/refresh` | refresh token kept in the Keychain, this device only, never synced |

Access tokens live 15 minutes; refresh tokens 30 days, rotated on use. Passkeys as a
second factor are planned.

**New-device rule (Live, build 72).** A device the account has never seen gets no private
data until a device that already holds the share approves it, or the athlete enters
the recovery code. Every new sign-in notifies every device (the Teammate notice is planned).

## 2. The crypto module (Live, build 72; key rotation and revocation planned)

What it stores and where:

| Item | Where | Attributes |
|---|---|---|
| Device key pair, P-256 | Secure Enclave | biometry or passcode required for use; the private key never leaves the enclave |
| Athlete share, 32 bytes | Keychain | accessible after first unlock, synchronizable, no biometric access control, so iCloud Keychain carries it and background Web runs can read it |
| Account key pair, P-256 | derived, never stored | private scalar derived from the share by rejection sampling: `HKDF-SHA256(share, salt "leave:v1:account-key", info accountId ‖ ":" ‖ counter)`, counter from 0 as ASCII, accepted when in `[1, n-1]`, else retry; stable across the account's devices |
| Recovery blob | server only | the share encrypted under an Argon2id-derived key from the recovery code; the app never stores the code |
| Refresh token | Keychain | this device only, not synchronizable |

Key roles, so the design is read correctly: the **share** is the syncable wrapping
secret. The **account key pair** is derived from it and is what other accounts (the
Teammate) encrypt to. The **Secure Enclave key** is the per-device identity that signs
device approvals and destructive-action confirmations; it is not a wrapping key,
because Secure Enclave keys cannot sync and a Keychain item with a biometric access
control cannot sync either.

**The share, plainly.** On iPhone the share is a Keychain item with
`kSecAttrSynchronizable` set, accessible after first unlock: the Leave app can read it
once the phone has been unlocked since it last restarted, and iCloud Keychain syncs it
if the athlete uses iCloud Keychain. It has no biometric access control, on purpose: a
changed face or a reset passcode must not lose the athlete's data, and a phone the
athlete approves or a recovery code must be able to carry it. Face ID guards the
actions (delete, export, changing the Teammate, unlocking for an AI app), not the share
itself. On Android the share is a file sealed by a key in the phone's hardware-backed
Keystore, with no StrongBox requirement and no user-authentication binding, for the
same reasons. The keys bound to biometrics are the device keys. The app lock (Face ID
at launch and after a minute away) is a screen in front of the app, not a second lock
on the share.

The package's public API:

```
createAthleteIdentity()        -> share created, SE key created, device public key
devicePublicKey()              -> P-256 public key, registered with the server per device
accountPublicKey()             -> P-256 public key derived from the share, registered once
wrapDataKey(dataKey, recordID) -> share-wrapped blob  (key = HKDF-SHA256(share, salt recordID, info "leave:v1:dek-wrap"); AES-256-GCM; AAD "leave:v1:<type>:<recordID>")
unwrapDataKey(blob, recordID)  -> dataKey (in memory only)
newDataKey()                   -> 32 random bytes
encryptRecord / decryptRecord  -> AES-256-GCM, wire nonce(12) ‖ ciphertext ‖ tag(16), AAD "leave:v1:<type>:<recordID>"
makeGrant(serverRsaPub, sessionID, exp, keys) -> grant (32-byte session key sealed with RSA-3072-OAEP-SHA256 to a key held in the cloud HSM; each data key sealed under the session key with AAD "leave:v1:grant:<sessionID>:<recordID>"; exp at most 15 minutes)
unwrapPending(accountWrappedDek, recordID) -> dataKey (HPKE open with the account private key, info "leave:v1:pending:<recordID>"); the app then share-wraps it and PUTs it back
wrapForAccount(dataKey, accountPub, recordID) -> HPKE blob (DHKEM P-256, HKDF-SHA256, AES-256-GCM, info "leave:v1:xwrap:<recordID>"); used for the Teammate
wrapShareForDevice(share, devicePub, approvalRequestID) -> HPKE blob (info "leave:v1:device-approve:<approvalRequestID>")
signConfirmation(action, targetID, timestamp) -> Secure Enclave signature for the confirmation header
rotateShare(progress)          -> (planned) new share; re-wraps every share-wrapped key the server lists, uploads the new blobs, then swaps the share
revokeShare()                  -> (planned) deletes the share locally and from iCloud Keychain (the server holds it disabled for 24 hours for undo)
makeRecoveryCode()             -> 28-character Crockford base32 code shown once; returns the Argon2id-encrypted share blob for the server
restoreFromRecoveryCode(code, blob) -> share
```

Wire details fixed in the API contract v1.2: the stored server blob is
`KMS(shareWrappedDek)`, nested, so the server's own service accounts cannot read a
record without the athlete's share; the app never sees the KMS layer. Records the
server creates (Web threads, Talk receipts, pitch drafts, exports, notes added from an
AI assistant) arrive as pending, wrapped with HPKE to the account public key; the app
unwraps them with the key derived from the share, re-wraps them with the share, and
returns them, after which only the share can open them. HPKE output is
`enc(65-byte uncompressed P-256 point) ‖ ciphertext`, with binding through the info
string and an empty HPKE AAD. Signatures are P-256 ECDSA, raw `r ‖ s` or DER.
Record types for AAD: strand, contract, contractField, contractMarks (from build 78: the
marks the phone keeps on a contract), file, embedding, thread, pitch, receipt, export.

Every destructive action (export, account deletion and its cancel, removing a device,
inviting or removing a Teammate, a guardian's invite, withdrawal and athlete deletion,
giving or withdrawing consent, and turning on private sharing for an AI app) first
evaluates the Secure Enclave key with the system biometric prompt (Face ID or the
passcode). Rotate and revoke are planned. When the phone is not approved, an emailed
code confirms account deletion.

Argon2id parameters: memory 64 MiB, 3 iterations, parallelism 1, 16-byte salt,
32-byte output. Recovery blob: `0x01 ‖ salt(16) ‖ nonce(12) ‖ AES-256-GCM(share) ‖
tag(16)`, base64url. HPKE comes from Apple's CryptoKit, which sets the app's minimum
iOS version at 17.

The package is published as its own repository at
github.com/LeaveYourAgent/leave-crypto (Apache-2.0), with test vectors at
`TestVectors/v1.json`; the server's test suite loads the same file, so both sides are
checked against the same values.

## 3. Screens that change or appear

1. **Sign-up.** Live (build 72): who the account is for, names and the athlete's date
   of birth as a neutral age gate (under 13 is refused with a plain explanation; 13 to
   17 requires a parent or legal guardian, who becomes the required Talent Teammate;
   18 and over holds the account), then a clickwrap agreement with an unchecked box and
   the linked Terms and Privacy Policy, with the accepted version recorded, then the
   plan gate. The key is created on the phone at sign-in with no step of its own. A
   recovery code is made later under Profile › Recovery code, and Home mentions it
   once. The AI consent sheet names Google's Gemini and Document AI and what each
   feature sends, and asks before anything is sent. Planned: a panel naming every
   vendor.
2. **Approve this device.** Live (build 72). After a code is verified on an unknown device: a
   short device code, "open Leave on a phone you already use and approve", and "use my
   recovery code instead". Until then the app shows public data only.
3. **Face ID lock.** Live (build 72), on by default: on launch and on return from background
   after 60 seconds; a blur replaces the app-switcher snapshot; private screens block
   screenshots (every screen classified, with a test, from build 78); the clipboard is
   cleared 30 seconds after a copy of private contract text and 2 minutes after a copy
   of a Talk reply (build 78); third-party keyboards are refused while a secret field
   (the sign-in code, the recovery code) is up (build 78).
4. **Your key.** Under Profile. Live (build 72): status (created date, devices holding
   the share, recovery code set or not). Planned: Rotate key, Revoke key with a 24-hour
   undo and "Export data first", and Delete account running Revoke first (today Delete
   account does not run Revoke; it deletes the records and keys on the server). All
   behind Face ID.
5. **Connections.** Live (build 72). Connecting an AI assistant happens on the web and grants
   public data only; the app never takes part in the sign-in. The screen offers one add
   row per supported assistant (Claude, ChatGPT, Gemini, and others as they support
   custom connectors), each with a fallback card holding the address and three steps.
   Connected assistants show their scopes, last use, unlock state, and a one-tap
   disconnect. **Share private data** shows the assistant's data-retention notice word
   for word, then Face ID, then the device-signed scope change and a grant for 15
   minutes (default) or 1 hour; **Lock now** ends it. Athletes aged 13 to 17 never see
   this control, and the server refuses it for them.
6. **Unlock notification.** Live (build 72). When an AI app asks for private data that
   is locked, Leave sends a notification naming the app. A tap opens Connections and
   goes straight to Face ID or the passcode, then a one-hour grant. Nothing happens if
   sharing is off for that app. There is no Unlock button on the notification and no
   silent background renewal. A "connected" notification refreshes the list and doubles as the security
   alert "Leave was connected to ChatGPT. Not you? Disconnect."
7. **Add to Leave.** Live (build 72), with the system camera and file pickers. On the
   phone, before any upload: text from PDFs and Word files, and on-device OCR for
   photos. Then the original file and its text, each encrypted with a fresh data key,
   go straight to storage through a short-lived signed URL. If on-device OCR
   confidence is low, the app asks before sending the scan to Google's Document AI,
   naming the provider. The phone checks 25 MB and 60 pages before upload. The server
   allows 10 uploads a day (UTC day; a contract with its text counts two). Processing
   and the queue show the server's real progress.
8. **Cloud pickers.** Planned. Google Drive, OneDrive and Dropbox run entirely on the
   phone through the system web-auth session; the token lives in memory only and the
   server never sees it.
9. **Talk to Leave.** Live (build 72) against the API: the reply streams, the phone
   holds the recent turns and a rolling summary, and from build 72 conversations are
   kept on the phone, encrypted, and Leave's servers don't keep them. From 1.3 (build 78)
   a reply that needs one contract's full terms, or why one Web thread fits, asks the
   phone for that one item's key mid-turn and the phone answers with a grant for that
   record alone. Each change Leave saves arrives as a receipt with an undo that lasts 24 hours.
   The app writes a report as a tagged PDF (PDF/UA-1) on the phone; the server's report
   kinds were switched off for part of 2026-10-25 while a fault was fixed and are on again since 2026-10-25.
   Voice (build 25) is recognised on the phone only; there is no server speech service.
   Text-to-speech is off by default for private content.
10. **Web matching in the background.** Planned. A background refresh (or a
    content-available notification) builds a grant with the strand keys only, runs the
    match, and completes any pending share wraps.
11. **Plan and usage.** Live (build 72): allowances used this month on the Plan page.
    When the Talk allowance and credits run out, Talk says so and points to Plan. In
    the iPhone app the plan and credit packs are bought through Apple in-app purchase
    (StoreKit 2), and the server grants them from Apple's signed transaction. In the US
    App Store the website's Stripe checkout is offered beside it, never in its place.
    Only the person who holds the billing sees purchase buttons. A parent or guardian
    buys for an athlete aged 13 to 17; the athlete and a Teammate see none.

## 4. Native modules

| Module | Status | Backs |
|---|---|---|
| Crypto bridge | Live (build 72) | everything in section 2, including recovery (Argon2id) and the Teammate wrap; key rotation and revocation are planned |
| App lock | Live (build 72) | the lock cover and its Face ID or passcode prompt, screenshot protection, clipboard expiry; turning the lock off takes Face ID or the passcode, and the lock cover and the app-switcher cover are drawn above alerts (build 45) |
| Document text | Live (build 72) | PDF extraction, on-device OCR with confidence, Word files |
| Speech | Live (build 72) | on-device recognition only, with an availability check; audio never leaves the phone |
| Secret fields | Live (build 78, iPhone only) | third-party keyboards refused while the emailed sign-in code or the recovery code field is up; the system keyboard stays. Android: no block; the recovery field is flagged for no personalized learning |
| Background refresh | Planned | scheduled Web matching |
| Attestation | Live (build 72) | App Check with App Attest; the token rides on every API call in X-Firebase-AppCheck, report-only on the server until it enforces |
| Universal links | Live (build 72) | leaveyouragent.com/billing/* only (the return from Stripe Checkout and the portal), plus the leaveyouragent:// scheme as the fallback page's button; the live app only. A link opens the Plan page and proves nothing: the plan changes only when Stripe's webhook reaches the server. The sign-in and connector hosts are deliberately excluded so the app can never take over a sign-in page |
| Push | Live (build 72) | Firebase Cloud Messaging through Swift Package Manager; the demo build carries no Firebase config and never registers; notifications carry one fixed sentence and record ids only, never private content; the token is taken off the account at sign-out |
| Crash reporting | Live (build 72) | Leave's own; no crash-reporting company receives anything. A report holds the error's kind, code locations as numbers, the app build, the iOS version and the iPhone model; no message text, names, logs or identifiers. Saved on the phone, sent at the next launch after a sign-in to Leave's API, kept 30 days. Off in Leave Demo; a switch in Profile. From build 54 the Crash reports screen shows the last report as plain rows (what kind, where as a count of places in Leave's code, how many times, the app and iOS versions, the iPhone model) with the exact report one row away. The phone keeps, for itself, the time it last sent a report so that the screen can say when; that time is never added to a report or sent, and it is removed when the switch is turned off or the account is deleted |
| Events | Planned | feature events to Leave's own endpoint under a random per-install identifier that is never joined to the account |

## 5. Decisions recorded (all 2026-09-23)

- **Minimum age 13, no exceptions.** The age gate refuses a birth date under 13. Ages
  13 to 17 use Leave through a parent or legal guardian, who is the required Teammate
  and holds billing. Live (build 72).
- **Key roles.** Share syncs and wraps; account key pair is derived from it; Secure
  Enclave key is per-device identity. Live (build 72).
- **Share accessibility.** Readable after first unlock so background Web runs work,
  with the trade-off recorded in the backend plan. Live (build 72). Background Web
  runs are planned.
- **Connecting an AI assistant is web-only and public-only.** Live (build 72). Private
  sharing is turned on in Connections with a signed confirmation and Face ID; it
  starts unlocked for 15 minutes, and later unlocks are 15 minutes or 1 hour; the
  control is hidden for athletes aged 13 to 17 and for their guardian's sign-in.
- **Assistants named in the app** are ChatGPT, Claude and Gemini. Live (build 72).
- **Terms version.** The app records the accepted version (currently 2026-10-05).
  Live (build 72): the version is recorded. Asking for re-acceptance when it changes
  is planned.
- **Third-party keyboards (2026-10-24).** Refused only on secret fields (the sign-in
  code, the recovery code), not app-wide, so accessibility and other-language keyboards
  keep working in Talk and everywhere else. Recorded against OWASP MASVS-PLATFORM-1 as
  partial, accepted. Live (build 78); iPhone only, Android has no block.

## 6. Security review

An internal review on 2026-09-23 against OWASP MASVS v2 found no critical issues. Its
"fix now" items have since shipped and are marked Live above. Deferred to the first connected build:
real sign-in verification with attempt limits, the collected-data privacy manifest,
certificate pinning (done in build 21), App Attest and proportionate jailbreak awareness.

A second internal review on 2026-10-24, part of a review of the whole service against
OWASP ASVS 5.0, MASVS v2 and the API and LLM Top 10 lists, found the local store's
key length (fixed in build 78), screens missing from the screenshot protection (fixed
in build 78), server-supplied links opened without a scheme check (build 78 opens
`https:` links only, from one place in the code that a Semgrep rule guards), and
third-party keyboards unrestricted on secret fields (build 78). Still open from it, and
planned: pinning the server's session public key and the Teammate's key on the phone
rather than trusting what the server returns, a nonce in the device-signed confirmation
header, and jailbreak awareness. The review's findings and fixes are logged on the
security page as they land.

## 7. Build order (remaining)

1. ~~The crypto package with test vectors, plus the bridge~~ (done; published at github.com/LeaveYourAgent/leave-crypto, Apache-2.0).
2. Real sign-in routes behind the existing screens, device registration, new-device
   approval, recovery code, Face ID lock and screen protection.
3. Session grants and the API repositories; the mock data path stays behind a build
   flag so test builds can keep running on fictional data.
4. ~~On-device contract pipeline, signed-URL uploads~~ (done); cloud pickers.
5. ~~On-device speech~~ (done, without any fallback).
6. Background Web refresh.
7. Connections: add rows, Share private data, Lock now, the unlock and connected
   notifications.
8. Usage meter, soft stops, credits link-out.
9. ~~Push, attestation~~ (done); crash-report filter, events client.
10. ~~Publish the crypto repository~~ (done: github.com/LeaveYourAgent/leave-crypto,
    Apache-2.0); the app half of the security page.

Steps 1 to 3 gate everything else. An independent penetration test of the app and the
service is planned once Leave is funded, with findings and fixes summarized on the
security page.
