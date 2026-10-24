# Changelog

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
