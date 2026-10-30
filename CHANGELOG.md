# Changelog

## 2026-10-30, a sealed grant's 15 minutes is what Leave's servers enforce

The app document said a Web run's sealed grant "expires 15 minutes after the phone made it". That is what Leave's servers enforce; it is not true of a copy. Today, when work goes on after the athlete asks (reading a contract, a Web run, a Talk reply that opens one contract or thread), the sealed grant is stored for that time in the task queue or the database, and copies can remain in the database's recovery logs for up to 7 days; someone who also held Leave's cloud key could open those copies. The row now says "on Leave's servers", and the security page says the same. The architecture document already described the handoff. The fix is approved and planned: those copies will be sealed under keys that are destroyed within about a day, after which they can no longer be opened by Leave or anyone holding a copy.

## 2026-10-30, the metadata list names the disclosure window's rule

Beside the day a disclosure window ends, Leave's servers keep which state or association rule that window comes from (an id from the public rules dataset, such as a state's high school 72-hour rule). The architecture document's metadata list now says so, the same as the security page and the Privacy Policy.

## 2026-10-30, the metadata list names the renewal flag

The architecture document's metadata list now also says Leave's servers keep whether a contract renews on its own, beside its dates, for the renewal and notice reminders. That flag has been kept since 2026-10-22; the list left it out.

## 2026-10-30, 1.5.0 on both stores (iOS build 81, Android code 21) and state rules on a deal

- Status: iOS 1.5.0 (build 81) on the App Store and Android 1.5.0 (version code 21) on Google Play, both since 2026-10-27. The crypto files are unchanged (the `ios-81` and `android-21` tags are the same commit as `ios-80` and `android-20`).
- State rules on a deal, on since 2026-10-30 for builds 81 and 21. The app document has a row for them: where the athlete plays is answered as a question (no location permission; a parent or guardian sets it for an athlete aged 13 to 17), the rules are public data shipped in the app and refreshed without sign-in, and a deal's packet is written on the phone and shared only by the athlete.
- Three more dates are kept on Leave's servers unencrypted, beside the payment dates, so the reminders can go: the signing date, the day a disclosure window ends and the day the athlete marked a deal reported. The architecture document's metadata list now names them, and also names what it had left out since 2026-10-22: the brand named on each contract, deliverable due dates, when a contract and its exclusivity end, and notice days. Never an amount or a clause.

## 2026-10-27, the edge's free-text exception, said plainly

The architecture document's Cloud Armor row now says that some injection signatures are not applied to the one request shape that carries an athlete's own words (an AI app's tool call, and four api routes that take free text), and why. That exception existed before and was wider (the whole MCP host); on 2026-10-27 it was narrowed to those requests, after a per-field exclusion was tested and found not to reach JSON body keys.

## 2026-10-26, Neo4j over TLS

The architecture document said the connection from Cloud Run to the graph database inside the private network was not yet TLS and was being changed. Since 2026-10-26 it runs over TLS, required on the server side, and the sync job proves the connection on every run. The same day the app's sign-in tokens became bound to Leave's own client on the server, enforced; nothing changes for the athlete.

## 2026-10-26, 1.4.0 on both stores (iOS build 80, Android code 20) and web search in Talk

- Status: iOS 1.4.0 (build 80) on the App Store and Android 1.4.0 (version code 20) on Google Play, both 2026-10-26. The crypto files are unchanged (the `ios-80` and `android-20` tags are the same commit as `ios-78` and `android-19`).
- Talk can search the public web for a question about something outside Leave, on since 2026-10-26 for builds 80 and 20. The architecture document and the app document say what a search sends (one short topic written from the question, checked to hold no name, school, team or place, with the market and sport; never the question, a strand, a contract term, a thread or the summary) and what the phone keeps (the answer only in the conversation on the phone, sources as title and domain, never an address on disk or in a request).
- Also from 1.4.0: a Talk reply that is not whole after 120 seconds ends with a quiet line.

## 2026-10-25, Android 1.3.1 (code 19) on Google Play

Code 19 against 18, checked with the app session against the source and the signed bundle: code shrinking (R8) is on in the release build (libraries shrunk and renamed; Leave's own classes keep their names by design; the entry below that said shrinking was still off no longer holds from 19 on); the crypto files are unchanged, `android-19` tagged on the same commit as `ios-78`, and the pipeline's drift check matched; the sign-in code field is reachable by screen readers, with no change to how the code is handled; no new screens, keyboards or links. The app document's status says so.

## 2026-10-25, status: on the App Store and Google Play

The status lines in the README and the architecture document now say Leave is on the App Store for iPhone (since 2026-10-19) and on Google Play for Android (since 2026-10-24); they had still said Android was in review. The app document's status names the store's current Android version, 1.3.1 (code 19), not yet checked here. Nothing else changed.

Same day: the architecture document's payments section said billing was in testing on Stripe's test environment (status as of 2026-10-01); production has run on Stripe's live environment since 2026-10-05, and the status now says so.

## 2026-10-25, the MCP document names both apps

`docs/mcp.md` now says the Leave app is on the App Store for iPhone and on Google Play (US) for Android, and the Connections screen is in both. Where it said Face ID alone, it says Face ID or the phone's fingerprint or face unlock. Nothing about the MCP server changed.

## 2026-10-25, Android 1.3.0 (code 18) on Google Play

Android 1.3.0 reached Google Play the same day; the app code is the same on both platforms, so the build 78 entry below holds for code 18 with these differences, each checked with the app session: the local store moves to AES-256 with a 192-bit key under the phone's hardware-backed Keystore key; every screen is classified for screenshot protection, including sign-in, sign-up and deletion; links open only when `https:`; leave-crypto is tagged `android-18` and the Android pipeline refuses an archive whose crypto files differ; the CI scanners cover the Gradle lockfiles. No third-party keyboard block on Android (the iPhone rule is iPhone only); the native recovery-code field asks keyboards not to learn from it. Code shrinking (R8) is still off. The server's report kinds, switched off earlier that day while a fault was fixed, are on again since 2026-10-25.

## 2026-10-25, iOS 1.3 (build 78) on the App Store

The iOS document is brought up to the store build, each fact checked against the build 78 source (the commit tagged `ios-78` in leave-crypto). Android 1.3 is not in production; nothing here is about it.

- Local store: the key is now AES-256 with a 192-bit random key. Builds 77 and before opened the store with the library's default AES-128, which read only the first 16 bytes of a key written as 32 hexadecimal characters, so 64 random bits protected it. Build 78 moves the stores to a new file under the new key on first launch and deletes the old files and key. The store library's encryption has no integrity check at any key length; the store was never in a backup and never left the phone.
- Screens: every screen is classified private or public with a test; saved conversations, reports, export, upload, the Teammate and Devices screens, the NIL payment screens and the sign-in, sign-up and deletion screens are now kept out of screenshots.
- Keyboards: a third-party keyboard is refused while a secret field is up (the sign-in code, the recovery code); everywhere else other keyboards stay (decided 2026-10-24, so accessibility and other-language keyboards keep working in Talk). Recorded against MASVS-PLATFORM-1 as partial, accepted.
- Clipboard: a copied Talk reply leaves the clipboard after 2 minutes. Contract text still leaves after 30 seconds.
- Links: server-supplied links open only when they are `https:`, from one place in the code that a Semgrep rule guards.
- Talk 1.3: a reply that needs one contract's full terms, or why one Web thread fits, asks the phone for that one item's key mid-turn; the phone answers with a grant for that record alone. The app writes a report as a tagged PDF (PDF/UA-1) on the phone; the server's report kinds were switched off for part of 2026-10-25 while a fault was fixed and are on again since 2026-10-25.
- Crypto: `contractMarks` is a new record type in the encryption's associated data. leave-crypto is tagged per store build (`ios-72`, `ios-77`, `ios-78`, `android-17`, `android-18`) and both release pipelines refuse an archive whose crypto files differ from the tagged copy.
- Checks on every change now include OSV-Scanner over every lockfile, a 7-day minimum age for new package versions, Semgrep with Leave's own blocking rules, and a PDF/UA-1 check of every report PDF with veraPDF.
- The usage row said purchases were web-only; it now matches item 11 (Apple in-app purchase beside the website's Stripe checkout on the US App Store).
- Section 6 records the 2026-10-24 internal review: what build 78 fixed, and what is still planned (pinning the server's session key and the Teammate's key on the phone, a nonce in the confirmation header, jailbreak awareness).

## 2026-10-24, further corrections

More lines checked against the code and the live configuration the same day:

- Architecture: section 2c describes how a still-sealed grant travels with work that continues after a request (reading a contract, a Web run, Talk's hand-off to a tool), including the Talk table row that lives at most 60 seconds.
- Architecture: deploy images carry a Binary Authorization attestation signed with cosign using a Cloud KMS key by the GitHub Actions deploy workflow, through Workload Identity Federation with no stored key. The document had said Sigstore keyless signing with SLSA provenance.
- Architecture: Grounding with Google Search is used only in an athlete's on-demand Web run; the crawler's nightly grounded crawl is paused since 2026-10-19. The model settings are named (`LEAVE_MODEL`, `LEAVE_MODEL_ROUTES`).
- MCP: the consent step is WorkOS's standard consent page with Leave's branding, naming the AI app; the sentence the document quoted was from the design and was never shown. Passkeys as a second factor are planned. A restart, a deploy or a second instance can mean the next private read asks again. The Connections screen is on the App Store since build 68 (2026-10-19), and the sheet that names strands and a thread's reason since build 72 (2026-10-22).
- MCP: a thread's reason now has instruction-like sentences stripped, as web text does, and is labelled as written by Leave's AI; contract terms are kept word for word and labelled as quoted text; the server's instructions tell the AI that contract terms, a thread's reason and strands are information, never instructions. Calls from a sign-in with no usable Leave account are limited per network address, 60 a minute and 300 an hour.
- iOS app: sign-up, identity, the crypto module, the unlock notification, Add to Leave, attestation, universal links, push and crash reporting are marked Live for build 72 (identity since build 68). Plan and usage describes in-app purchase through Apple beside the website's Stripe checkout in the US App Store. The list of actions confirmed with Face ID or the passcode is complete. Key rotation and revocation are marked planned, in the crypto module's API too. The recorded Terms version is 2026-10-05.

## 2026-10-24

An internal security review compared these documents with the code. Corrections:

- Architecture: Firebase App Check is report-only today. The app sends an App Attest or Play Integrity token on every request and the server records failures; the document had said it was enforced.
- Architecture: the status is as of 2026-10-24, with the iPhone app live and the Android app in review on Google Play, not yet in production. Two lines, how grants are stored for work that continues after a request and how deploy images are attested, are being reviewed and will be updated.
- Architecture: Rotate and Revoke key are designed, not built. Delete account deletes the records and their keys without a separate Revoke step; the document had said it ran Revoke.
- Architecture and MCP: Google's abuse-monitoring retention is up to 90 days, for flagged prompts only (the documents still said up to 30 days). The models in use are named: Gemini 4 Argon for Talk and contract reading; Gemini 3.8 Flash for the Web run, long Talk inputs and side-by-side reports. The documents had said every call ran on Gemini 3.8 Flash.
- Architecture: the Teammate's new-sign-in notice and passkeys as a second factor are planned, not shipped. The CMEK key scope, the dependency controls and the test setup now match the code.
- Architecture: inside the private network, the connection from Cloud Run to the graph database (Neo4j) is not yet TLS; that is being changed, and the document now says so.
- MCP: since 2026-10-22, a connection turned on under the sheet that names strands, the private reason behind each thread and contract terms can read all three while unlocked, including the new `get_strands` tool. The document had said strands and reasons were not readable from an AI app.
- MCP: what is filtered today is stated: hidden characters from everything, and instruction-like sentences from web text. Contract terms and a thread's reason are not yet labelled as quoted text for the AI; that is planned.
- MCP: an unlock is held by the one server instance that received it; when more than one is running, a read can answer "locked" and ask the phone again.
- iOS app: the status is as of 2026-10-24, with store build 72. Features in that build are marked Live, the permission list is complete (Face ID and notifications were missing), and Talk's saved conversations are kept on the phone.
- iOS app: the client crypto package is listed as published at https://github.com/LeaveYourAgent/leave-crypto, recovery is listed as live, and Talk conversations are kept on the phone, encrypted, from build 72; Leave's servers don't keep them.

## 2026-10-20

- The apps' client cryptography is published at https://github.com/LeaveYourAgent/leave-crypto under Apache-2.0: the iPhone and Android cores, the bridges that hold the key share, the shared test vectors, and a cross-check of the iPhone, Android and a test copy of the server code.

## 2026-10-19

- Leave is on the App Store for iPhone; Android is in review with Google Play. The README's status is updated, and the security page (https://leaveyouragent.com/security) now marks what is live.
- Not yet done, and said so: the client crypto module is not yet open source. The architecture document said it was published; it now says it is to be published.
- Since 2026-09-25 and recorded on the security page: Gemini 4 Argon reads contracts and answers Talk (Gemini 3.8 Flash for the rest); the nightly grounded web crawl is stopped and what it stored is deleted; a Web run now searches live with topics checked to hold no name, school or team; Google's abuse-monitoring retention is now up to 90 days for flagged prompts only.

## 2026-09-25

- Every model call moved from Claude Opus 5.5 (Anthropic, served by Vertex AI) to Gemini 3.8 Flash on Vertex AI. Anthropic no longer processes Leave data. The crawler uses Grounding with Google Search, with no athlete data in the prompt. A thinking level per route replaces Claude's effort setting. Google's up-to-30-day abuse-monitoring retention still applies and is disclosed; Google does not train on the data.

## 2026-09-23

- leaveyouragent.com: DNSSEC turned on, CAA records added (Google Trust Services, Let's Encrypt, plus Cloudflare's issuers), DMARC moved from none to quarantine, domain submitted to the HSTS preload list.

- Repository published: architecture and security plan, API contract, MCP design, iOS app security plan.
- First transparency report: zero requests.
- Security page published at https://leaveyouragent.com/security.
