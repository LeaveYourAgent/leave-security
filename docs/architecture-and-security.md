# Leave on Google Cloud: architecture and security

Design, not yet built — status as of 2026-09-23.

**What exists today**

- **Live:** the marketing site, the Terms and the Privacy Policy at leaveyouragent.com.
- **Configured:** the WorkOS production environment (custom domain auth.leaveyouragent.com verified, client ID metadata documents and dynamic client registration on).
- **Written, not applied:** the Terraform for the three stages, validated but not yet applied to Google Cloud.
- **In progress:** the Node backend workspace is being scaffolded; the iOS app plan is written and the app is in development.
- **Design:** the MCP server (see `mcp.md`), and everything else in this document unless a line says otherwise.

Written 2026-09-22, security review 2026-09-23. One environment (production) for the October 1 beta. Companion documents: `api-contract.md` (the routes the app calls and the cryptographic framing) and `mcp.md` (Leave inside ChatGPT, Claude, Gemini and other MCP-capable apps).

**This document is public by design.** Leave publishes its infrastructure and security plan so athletes and anyone reviewing it can see how their data is handled, and keeps it accurate as the system changes. Nothing in it is secret: no keys, credentials, internal IDs or customer data appear here, and the controls described do not depend on the design being hidden. Commercial figures (vendor prices, unit costs, budgets and margins) are left out of the public edition; they do not change any control described here.

## 1. What the app needs

| Product surface | What it does | Infrastructure it implies |
|---|---|---|
| Account + sign-in | Emailed sign-in code (no password) for the app and for the MCP sign-in page. Adults hold their own account with one Talent Teammate seat. Athletes aged 13 to 17 are invited by a parent or legal guardian, who is their required Teammate. No one under 13 | WorkOS AuthKit with Magic Auth (the emailed six-digit code). The WorkOS iOS SDK expects a hosted page, so the API fronts it with `POST /auth/code` and `POST /auth/verify` (server-side WorkOS calls) and the designed Email and Code screens stay. Postgres for accounts/seats/ACLs keyed by the WorkOS user ID |
| "Hey Leave" in ChatGPT, Claude, Gemini and other MCP-capable apps | Remote MCP server. Connecting happens on the web, signing in with the same emailed code as the app, and grants public data only | MCP served at the root of mcp.leaveyouragent.com inside the `api` service (MCP spec 2026-07-28, stateless, SDK v2), as an OAuth resource server; WorkOS AuthKit is the OAuth 2.1 authorization server (client ID metadata documents and dynamic client registration, PKCE, protected-resource metadata, resource indicators) on auth.leaveyouragent.com. Full design in `mcp.md` |
| Talk to Leave (voice + type) | Live transcript, change receipts with Undo, on-device speech where available, spoken replies optional | On-device iOS speech with Chirp 3 streaming as an opt-in fallback; Text-to-Speech Chirp 3 HD for "read connections aloud"; Claude Opus 5.5 for the conversation |
| Contracts | Upload PDF/DOC/DOCX/JPG/PNG/HEIC up to 25 MB / 60 pages from camera, Files, cloud pickers, or the AI chat; "Done in 41 seconds"; six-part explanation; brand colors + logo | The phone extracts the text first; Cloud Storage for the encrypted original, Cloud Tasks queue + Cloud Run worker (LibreOffice + HEIC conversion in the container), Document AI Layout Parser only for scans the phone cannot read (the athlete is told), Claude Opus 5.5 on Vertex AI with structured output to extract the six parts and flagged clauses, embeddings into pgvector, nodes and edges into Neo4j, Brandfetch Brand API |
| Files | 5 GB per athlete, contracts shared with the Teammate, everything else athlete-only, delete removes derived data; cloud imports (Drive, OneDrive, Dropbox) run on the phone | Cloud Storage with per-object metadata + signed URLs, Postgres ACL rows, KMS envelope keys |
| The Leave Web | Private strands + public record + public signals (brand posts, casting calls, press, deal announcements, sponsor programs, own contracts); each thread = strand + signal + potential move; blocked-by-exclusivity edges with re-open dates | Neo4j knowledge graph (athletes, strands, signals, contracts, clauses, brands, markets and the edges between them), pgvector embeddings for candidate matching, Cloud Scheduler + Cloud Run Jobs for the signal crawlers, Claude web search for discovery, Claude Opus 5.5 for connection generation, FCM push for "new connection" |
| Profile | Public record from the talent database, private card, Teammate, Plan (Stripe portal), Export data, Connections (MCP clients) | Cloud SQL `talent` table holding records only for athletes who have signed up (the public record is looked up at sign-up; no profile is ever pre-built for someone who is not a user), Stripe Billing + webhooks, export job writing to Cloud Storage, MCP client table |
| Learn series, Up Next, Accessibility | Static content, reminders, settings | App bundle + Postgres; Cloud Scheduler for payment/deliverable reminders |
| Analytics events | tour_started, tour_step_viewed, etc., never attached to private data | First-party events table in Postgres, pseudonymous, exported to BigQuery in Leave's own project (Firebase Analytics dropped in the security review) |
| Mobile release | iOS and Android, push, crash reports | Firebase for FCM, Crashlytics and App Check only (no Firebase Auth, no Firebase Analytics), Apple Developer Program, Google Play |

Not needed on Google Cloud: Firebase Authentication or Identity Platform (WorkOS is the identity provider), Memorystore Redis at launch (Postgres covers rate-limit counters and job dedupe at this scale), GKE (Cloud Run does everything here), a separate vector database product (pgvector on Cloud SQL holds every embedding and is enough well past 10,000 athletes).

### Data layer, decided

- **Postgres on Cloud SQL is the system of record.** Accounts, seats, the talent table, contracts and their extracted fields, strands, signals, files, the list of connected MCP clients per athlete (mirrored from WorkOS for the Connections screen), plus a `vector` column (pgvector) on every row that gets embedded.
- **pgvector holds the embeddings.** Strands, signals, contract clauses, talent rows and connections are embedded with `gemini-embedding-001` on Vertex AI when written. Embeddings of private text are generated inside a session grant and stored encrypted (section 2c); the HNSW index covers public signals only. Matching starts as a nearest-neighbour query, which produces the candidate pairs the graph and the model then reason over.
- **Neo4j holds the knowledge graph.** Nodes: Athlete, Strand, Signal, Contract, Clause, Brand, Market, Program. Edges: SAID, HAS_CONTRACT, HAS_CLAUSE, BLOCKS (with the clause end date), MENTIONS, IN_MARKET, CONNECTS. The Web screen, the "why it fits" evidence, and the blocked-until-August edges are graph queries. Neo4j is a projection rebuilt from Postgres (an outbox table plus a small sync job), so losing or resizing it never loses data. It runs self-hosted (Neo4j Community) on a Confidential VM inside the VPC with a CMEK disk and no public IP (review finding 6).
- **Contract pipeline:** the phone extracts text first (PDFKit for PDFs, on-device DOCX conversion, Apple's Vision OCR for photos) and uploads Markdown plus the encrypted original → only for scans the phone cannot read, and only after the athlete is told and agrees, the worker sends the file to Document AI Layout Parser, which returns Markdown with headings, tables and page anchors → Claude Opus 5.5 with structured output extracts term, deliverables, usage rights, payment dates, exclusivity, termination, plus flagged clauses with plain-English notes → rows and embeddings into Postgres → nodes and edges into Neo4j → Brandfetch colors and logo → "Pending your OK".

## 2. Organization layout

Google's landing-zone guidance is organization → folders → projects, IAM at folder level, everything in Terraform. Sized for one founder and one environment, with room to add staging later without moving anything:

```
Organization  leaveyouragent.com        (already exists: created by the Google Workspace account)
│
├── Billing account                     (budget and forecast alerts)
│
├── Folder  bootstrap
│     └── Project  <seed-project>       Terraform state bucket, Workload Identity pool for GitHub Actions,
│                                        Artifact Registry, org-level log sink destination
│
├── Folder  prod
│     └── Project  <prod-project>       everything that serves athletes (also the Firebase project for push, crashes, App Check)
│
└── Folder  nonprod                     (empty until staging is wanted; same Terraform module, different tfvars)
```

Why two projects instead of one: the seed project holds the things that must outlive or rebuild prod (state, CI identity, images) and gets a tighter IAM policy. Why not more: every extra project is another place to configure logging, alerts and IAM, and a one-person team gets nothing for it yet.

Region: **us-central1** (Iowa). Every service used here is available there, including Claude on Vertex AI and Chirp 3. No multi-region anything at launch; Cloud SQL backups and the state bucket use the US multi-region for durability only.

Domains: DNS and the marketing site stay at Cloudflare. DNS-only (grey cloud) records for `api.leaveyouragent.com` and `mcp.leaveyouragent.com` point at the Google load balancer with Google-managed certificates.

### Identities and access

- The organization resource already exists because leaveyouragent.com runs on Google Workspace; Workspace users are the org's identities, and no Cloud Identity setup is needed.
- Two human accounts: the founder's account as Organization Admin and Billing Admin with a hardware security key, and a break-glass admin account whose credentials live offline. 2-Step Verification is enforced org-wide.
- No user gets project Editor/Owner day to day; the narrow roles actually used are granted (Cloud Run Admin, Cloud SQL Admin, Secret Manager Admin), and Owner is used only through the break-glass account.
- One service account per Cloud Run service and job (`api`, `worker`, `crawler`, `scheduler-invoker`), each with only the roles it needs. No service-account keys anywhere; GitHub Actions deploys through Workload Identity Federation.

### Organization policies to set on day one

| Constraint | Setting | Why |
|---|---|---|
| `iam.allowedPolicyMemberDomains` | the Workspace customer ID (`<customer-id>`) | nobody outside the org can be granted access, even by mistake |
| `iam.disableServiceAccountKeyCreation` | enforce | keys are the number one leak vector; WIF replaces them |
| `iam.automaticIamGrantsForDefaultServiceAccounts` | enforce | stops default service accounts getting Editor |
| `storage.publicAccessPrevention` | enforce | contract files can never be made public |
| `storage.uniformBucketLevelAccess` | enforce | IAM only, no legacy ACLs |
| `sql.restrictPublicIp` | enforce | database is private-IP only |
| `compute.vmExternalIpAccess` | deny all | the one VM (Neo4j) has no external IP, and nothing else may get one |
| `compute.skipDefaultNetworkCreation` | enforce | the one VPC is created in Terraform |
| `gcp.resourceLocations` | `in:us-locations` | athlete data stays in the US |
| `run.allowedIngress` | internal-and-cloud-load-balancing | Cloud Run is only reachable through the load balancer and Cloud Armor |
| `essentialcontacts.allowedContactDomains` | leaveyouragent.com | security and billing notices go to a leaveyouragent.com address |

### Security baseline

- Cloud Audit Logs: Admin Activity is on by default; Data Access logs are also enabled for Cloud SQL, Cloud Storage and Secret Manager in the production project (who opened which contract is something Leave may be asked for).
- Security Command Center Standard tier at the org level.
- Secret Manager for every third-party key (WorkOS API key and client secret, Brandfetch, Resend, Stripe, Apple push). Claude runs through Vertex AI with the service account's own identity, so there is no Anthropic key to store. Cloud Run mounts secrets; nothing lives in env files.
- Cloud KMS: one key ring with an HSM key that wraps every per-record data key together with the athlete's own key share, so "not Leave staff" in the private card is true in the cryptography. Section 2c has the full design.
- Logging exclusions so request bodies, transcripts and strand text never land in Cloud Logging (the design promises "never logged").
- Cloud Armor Standard on the load balancer: preconfigured WAF rules, per-IP rate limits on `/auth/*`, `/oauth/*` and uploads.
- Firebase App Check on the mobile API so scripted clients cannot burn Claude tokens.
- WorkOS AuthKit issues every user token. The API verifies WorkOS access tokens (JWTs) with the published JWKS; the MCP server does the same and checks the audience (the resource indicator) so a token minted for another resource cannot be replayed against it.

## 2b. Firewall, rate limiting and abuse controls

There is no single "firewall" box in this design. Protection is layered, and each layer answers a different question.

| Layer | What it stops | How it is done here |
|---|---|---|
| Google's edge | Volumetric DDoS (L3/L4) | Always on for anything behind the external load balancer; nothing to configure |
| Cloud Armor Standard on the load balancer | Web attacks (SQL injection, XSS, path traversal, scanners), request floods from one IP, junk traffic to the OAuth and upload paths | Preconfigured WAF rule set; per-IP rate limits: 60 requests/min on `/auth/*` and `/oauth/*`, 10/min on uploads, 600/min elsewhere, and a flood guard of 1,200/min on the MCP host (ChatGPT and Claude call from shared egress IPs, so the per-athlete MCP limit lives in the app, per client); ban for 10 minutes on breach; log every rule hit |
| Cloud Run ingress | Anyone reaching the service by its `run.app` URL and skipping Cloud Armor | Ingress set to internal-and-cloud-load-balancing by org policy; `run.app` URLs answer 404 from outside |
| VPC firewall | Lateral movement inside the network | No default network, so the implied deny stands. Explicit rules: the Cloud Run egress subnet to Cloud SQL on 5432 and to the Neo4j VM on 7687, and nothing else. No SSH from the internet, no bastion; the Neo4j VM is administered through OS Login |
| Cloud SQL | Database exposure | Private IP only, no public address, no authorized networks, IAM database authentication, SSL required |
| Neo4j on a Confidential VM | Database exposure | Self-hosted Neo4j Community inside the VPC: private IP only, no external IP, CMEK disk, reachable only from the Cloud Run egress subnet, credentials in Secret Manager. It holds opaque IDs and encrypted properties only |
| Firebase App Check | Scripted clients calling the mobile API with a stolen token and burning Claude tokens | App Attest on iOS, Play Integrity on Android, enforced on every `api.` route the app uses. MCP traffic comes from the AI apps' servers (ChatGPT, Claude, Gemini and other MCP-capable apps), so it is covered by OAuth tokens and Cloud Armor instead |
| Application rate limits (per user, not per IP) | The abuse Cloud Armor cannot see because it hides behind a valid account or many IPs | Counters in Postgres (a `rate_limits` table with a token bucket per key). Limits below |
| Spend guardrails | A runaway loop or a scripted account turning into a large Vertex AI bill | Vertex AI and Document AI per-minute quotas set low; budget and forecast alerts; a daily Cloud Monitoring alert on Vertex usage over 2× the 7-day average |
| Crawler egress | Server-side request forgery from a crafted "public signal" URL | The crawler job refuses private IP ranges and metadata addresses, follows at most 3 redirects, caps response size, and has no VPC access at all |
| Secrets and identities | Leaked keys | No service-account keys (org policy), Workload Identity for GitHub, Secret Manager with a yearly rotation reminder, one service account per workload |
| Data | Exposure at rest, in a backup, in a vendor, or in a log | Two-part envelope encryption (Cloud KMS HSM key plus the athlete's key share, section 2c), CMEK on every store, uniform bucket access with public-access prevention, 15-minute signed URLs, Data Access audit logs, logging exclusions for transcripts and bodies |

### Per-user limits the app enforces itself

| Action | Limit | Why |
|---|---|---|
| Sign-in code requests | 5 per email per 15 min, 20 per IP per hour; code expires in 10 min; 5 wrong attempts invalidates it | stops email bombing and code guessing |
| OAuth dynamic client registration and token issuance | handled inside WorkOS AuthKit, with WorkOS Radar available for bot and abuse detection | the authorization server is not Leave's code, so these limits are not Leave's to build |
| Talk to Leave | 200 turns per athlete per day, 30 per hour; 4 concurrent streams | a normal day is about 20 turns; the cap contains a scripted or runaway account |
| Contract uploads | 10 per athlete per day, 25 MB and 60 pages each; Cloud Tasks queue capped at 20 dispatches per second | keeps Document AI and the model within quota and makes a queue backlog visible |
| Pitch drafts and explanations | 50 per athlete per day | same reason |
| Teammate invites | 5 per athlete per day | one seat exists; repeated invites are abuse |
| Export data | 2 per athlete per day | each export is a job |
| Cloud picker imports | 100 files per import, 5 imports per day | the design promises one-shot access, so there is no standing link to abuse |
| MCP tool calls | 60 a minute and 600 a day per connected client | see `mcp.md` |

### Plan allowances and hard caps

The Base plan is $99 a month on the web, with a 30-day trial. It includes a monthly allowance, and the gateway enforces hard caps regardless of anything the athlete has bought.

| Allowance per month | Included |
|---|---|
| Talk to Leave | 300 talks |
| Contracts | 5 uploads, 100 pages |
| The Web | unlimited threads; matching runs up to 4 a day |
| Pitch drafts | 30 |
| Leave speaking | 120 minutes |
| Files | 5 GB |

- **Trial:** half allowances (150 talks, 2 contracts, 60 speaking minutes).
- **More usage: Leave credits**, 1,000 credits for $10, bought on leaveyouragent.com through Stripe; the app links out to the web. Auto-refill is off by default and, when turned on, has a monthly cap the athlete sets. Credits do not expire while the subscription is active. Unused credits are forfeited 30 days after the subscription ends, except where the law requires a refund (published in the Terms).
- **Athletes aged 13 to 17:** only the parent or legal guardian can buy.
- **What the athlete sees:** usage in Profile → Plan ("This month: 142 of 300 talks · 2 of 5 contracts · 34 of 120 speaking minutes · 250 credits"), reset on the billing day. At 80%, a push and an in-app line. At 100%, a soft stop: a Talk conversation always finishes its current reply, then offers "Add credits" or "Wait for the 1st"; contract upload shows the same choice before processing starts, never after.
- The Teammate seat draws from the athlete's allowance (their only AI action is "Ask about this contract").

Hard caps the gateway enforces regardless of credits:

| Cap | Value | Purpose |
|---|---|---|
| Daily tokens per account | 2M input, 200K output | a runaway loop or scripted client is contained to one bad day |
| Monthly AI usage per account | a ceiling support can raise | credits cannot be used to burn through usage faster than intended |
| Concurrent model calls per account | 2 | |
| Contract size | 25 MB, 60 pages; Document AI runs at most once per file hash | |
| Context per Talk turn | last 20 turns plus a rolling summary, 12K tokens max | keeps each turn bounded as conversations grow |
| Effort per route | `low` for receipts and short replies, `medium` for Talk, `high` for extraction and Web connections | Opus 5.5 defaults to `medium`; effort is set explicitly per route |
| Global daily Vertex usage | alert at 2× the 7-day average, page at 4× | |
| Kill switch | a feature flag that drops effort to `low`, disables speaking, and pauses Web runs | an incident response that does not need a deploy |

Every model, Document AI, speech and embedding call writes one row to a `usage_events` table (athlete, feature, model, token and unit counts, request ID). Allowance counters are that table summed per billing period, checked before the call and decremented after. Private content never enters it.

### Things this design does not need yet

VPC Service Controls, Cloud IDS, Cloud NGFW Enterprise, Cloud Armor Enterprise, Cloud NAT, a bastion host, or Identity-Aware Proxy. IAP becomes the right answer the day there is an internal admin web app; it goes behind IAP rather than getting its own login.

## 2c. Encryption: private data readable by the athlete only

Requirement (2026-09-23): every piece of private data is encrypted so that no Leave employee, no Google employee and no vendor can read it. Only the athlete can, and the athlete holds a key they can revoke.

### What "encrypted at rest" does and does not give you

Google encrypts every disk by default, and CMEK puts that under a key Leave owns. That protects against a stolen disk. It does nothing against an employee with database access, because the database decrypts for anyone it lets in. "No employee can read it" is only true when the key needed to decrypt is something employees do not have. That means part of the key has to live with the athlete.

### The design: two-part key, sealed processing

Each athlete has two secrets with different jobs. The **athlete key share** (32 random bytes) is the wrapping secret: it is stored in the Keychain with `AfterFirstUnlock` accessibility and synced through iCloud Keychain (which Apple end-to-end encrypts), and it must not carry a biometric access control, because items with one cannot sync. A P-256 **account key pair** is derived from the share so the Teammate can be given contract keys. The **device key** is a P-256 key in the Secure Enclave (never exportable, biometrics or passcode required) and is the per-device identity: it signs device approvals and every destructive action. It is not a wrapping key, because Secure Enclave keys cannot sync. A one-time recovery code (28 characters, shown once, like Apple's) regenerates the share if every device is lost.

Every private item (a strand, a contract's text and extracted fields, a file) is encrypted with its own **data key** (AES-256-GCM, unique nonce per record, record ID as associated data). Each data key is wrapped twice, and both wraps are required to unwrap:

1. by the project's **Cloud KMS key** (HSM protection level), which only the `api` and `worker` service accounts may use, and
2. by the **athlete key share**, which the server never stores.

Employees, database backups, Neo4j, Cloud Storage and log exports therefore hold ciphertext only. Someone with full access to every Google resource still cannot read a strand, because the athlete share is not there.

### How Leave still does its job

Leave has to read strands and contracts to explain a clause or find a Web thread. It does so only inside a **session grant**:

- The athlete share never leaves the phone (review finding 1). When the app opens (or iOS background refresh fires), the app unwraps only the data keys the session needs and sends those, each bound to its record ID and the session, sealed under a session key that is itself sealed to a server public key whose private half lives in Cloud KMS (HSM). Any API instance can unseal through KMS, so no session affinity or stored state is needed; the data keys live in instance memory for at most 15 minutes and are never written anywhere. The server can decrypt what the athlete is using now, never the whole account. Exact shapes are in `api-contract.md`.
- Talk to Leave, contract explanations, pitch drafts and Web matching all run within a grant. Web matching runs when a grant exists rather than as a nightly server job; the athlete sees new threads when they open the app, and push notifications carry only public facts ("a new casting call in Dallas"), never private ones.
- Connecting an AI app happens on the web (the hosted WorkOS page inside the AI app's own sign-in sheet) and grants public access only; there is no in-app consent step, because an associated-domain universal link would let the installed app capture the OAuth page mid-flow and break sign-in. Private access is a per-client switch turned on later in the app, off by default, behind Face ID, with a notice naming that provider's retention policy (review finding 2). Athletes aged 13 to 17 cannot turn it on. Because an AI app cannot hold the athlete's keys, private reads over MCP work only while an unlock is live: 15 minutes by default or 1 hour at most, held in API memory only, with a push to the phone and a one-tap re-unlock when it lapses and a "Lock now" in the Connections screen. Public reads and all writes work at any time. The MCP server is mounted in the `api` service under the `mcp.` host, so it adds no new infrastructure.
- The Teammate sees contracts, so each contract's data key is also wrapped to the Teammate's public key. Private strands are wrapped to the athlete only, which makes the design's "never your Teammate, not Leave staff" true in the cryptography, not just in the access rules.

### What stays plaintext, on purpose

- **Public signals** (brand posts, casting calls, press) and their embeddings. They are public; the pgvector HNSW index is built over these only.
- **Private embeddings are ciphertext.** An embedding of a private strand can be inverted back to the sentence, so strand embeddings are stored encrypted with the data key and compared against the public index in memory during a grant. An athlete has tens of strands, so this is a few milliseconds.
- **Neo4j** holds private nodes with an opaque ID and encrypted properties. The graph shape is visible (this athlete has 9 strands and 4 threads); the content is not.
- **Metadata** the service needs to run: account IDs, seat state, contract count, payment dates, file sizes, timestamps. Support works from this and nothing else.

### The athlete's key screen (Profile → Connections, plan & data → Your key)

- **Status**: key created on date, devices holding it, recovery code set or not.
- **Rotate key**: generates a new share and re-wraps every data key on the device. Use after a lost device.
- **Revoke key**: deletes the share from the device and from iCloud Keychain, with a 24-hour undo window during which the share is held disabled. After that every private item on Leave's servers is permanently unreadable, including in backups, in Neo4j and in exports (crypto-shredding). The screen offers "Export data" first and says so in plain language.
- **Delete account** runs Revoke, then deletes the ciphertext and metadata.

### The AI layer is where the promise can break

Whenever Leave sends a strand or a clause to a model, the plaintext leaves the sealed session. Two facts from Google's and Anthropic's current terms shape the choice of model:

- **Claude Fable 5 and Fable 5.1 are "Covered Models": prompts and responses are retained for 30 days and shared with Anthropic for abuse monitoring, and this cannot be waived on Vertex.** Using Fable on private data would mean Anthropic staff could, under their trust-and-safety process, read it for 30 days. Leave does not use Fable for that reason.
- **Claude Opus 5.5 on Vertex is eligible for Google's abuse-monitoring exception (zero data retention)**, which also turns off the default 24-hour prompt cache. With the exception approved, nothing is stored after the response.

Decision (2026-09-23): **every model call runs on Claude Opus 5.5 on Vertex AI with the zero-data-retention exception approved before launch.** Opus 5.5 shipped on 2026-09-22, is in the Vertex Model Garden, and is not a Covered Model (Anthropic lists only Fable 5.1, Mythos 5.1, Fable 5 and Mythos 5). One model for everything, including the crawler. Speech-to-text runs on device; the Chirp 3 fallback does not log audio unless data logging is opted into, which it is not. Document AI processes a scanned contract and returns; it does not retain documents unless a dataset is configured, which it is not. Embeddings of private text are generated inside a grant and stored encrypted.

### Google-side controls, layered under the cryptography

- **CMEK** on Cloud SQL, its backups, the file bucket, the Neo4j disk and Artifact Registry with the same HSM key, so disabling one key shreds the whole environment if ever needed.
- **KMS IAM**: only the two runtime service accounts hold `cloudkms.cryptoKeyVersions.useToDecrypt`; an IAM deny policy blocks every human principal from that permission, and the org policy already forbids service-account keys. Changing the deny policy is an Admin Activity audit log entry that pages the phone.
- **Access Transparency and Access Approval** (they need a paid Google Cloud support plan): every access to Leave's data by Google personnel is logged, and each one needs Leave's approval first.
- **Data Access audit logs** on KMS, Cloud SQL and Cloud Storage, with the logging exclusions from 2b so no plaintext ever reaches Cloud Logging.
- **Not in the plan yet**: Cloud EKM with Key Access Justifications (keys held outside Google entirely) and Confidential Space (hardware-attested workloads that release KMS keys only to a measured container image). Both raise the bar against Google itself, but the athlete share already keeps Google out of the plaintext, and Cloud Run does not run on confidential hardware today. Revisit when a customer or investor security review asks for it.

### Trade-offs to be clear about

- Web matching happens when the athlete opens the app, not at 3 a.m. iOS background refresh narrows the gap but is not guaranteed.
- A user who loses every device and the recovery code loses their private data. That is the point, and the onboarding copy has to say it.
- Support cannot "take a look" at an athlete's strand or contract. They can see that a contract exists, its size and its state.
- Android needs the same share model on the Android Keystore with Google's key backup, and cross-platform recovery relies on the recovery code.

## 2d. Security review (2026-09-23): holes found and closed

Standard for this review: Leave is a security-first consumer company asking athletes to trust a developer they have never met. Wherever a choice trades convenience or cost against the athlete, the athlete wins. Findings are ordered by severity. Each one changed the plan; the affected sections above already reflect the fix.

### Critical

**1. The athlete share was leaving the phone.** Section 2c (as first written) sent the athlete's key share to the API for a 15-minute grant. That put the one secret that keeps every employee out of the plaintext into a shared Node process that serves 80 concurrent requests, and it meant a malicious or coerced deploy could exfiltrate it. Fix: **the share never leaves the phone.** The app unwraps only the per-record data keys the session needs (the strands for a matching run, the one contract being explained) and sends those keys, each bound to its record ID and to the session, with a 15-minute lifetime. The server can decrypt only what the athlete is using right now, never the whole account, and never anything wrapped after revocation. Rotation and re-wrapping run on the device.

**2. Private data flowed to AI apps under their retention rules.** The first MCP design let a connected AI app read Web threads, and a thread carries the strand ("you told Leave cookies are your go-to"). Once that text is in a ChatGPT conversation it is stored under OpenAI's consumer policy (kept until the user deletes it, and used for training unless the user opts out), which Leave cannot control; the same holds for Claude, Gemini and other MCP-capable apps under their own policies. Fix: **AI apps get the public record and public signals by default.** Private strands and contract text sit behind a separate per-client switch in the app, off by default, with a plain warning naming the provider's retention policy, and they are readable only during a live unlock of 15 minutes or 1 hour. Athletes aged 13 to 17 cannot turn it on. Writes are always allowed (the athlete can tell ChatGPT a private fact and it becomes an encrypted strand). Every connected app shows in the Connections screen with its access and a one-tap disconnect, and access tokens are resource-bound and last 5 minutes.

**3. Nothing stopped a bad deploy from running.** The org policies keep humans away from keys, but any image pushed to Artifact Registry would run. Fix: **Binary Authorization on Cloud Run** requiring a Sigstore (cosign) signature produced only by the GitHub Actions workflow through Workload Identity Federation, plus SLSA provenance. The workflow, the Dockerfiles and the deploy policy are public in the repo, so anyone can check that what runs is what was reviewed. This is the strongest available answer to "how do I know the code does what you say" short of confidential hardware, which Cloud Run does not offer.

### High

**4. Audit logs could be edited by the person they audit.** A solo founder is also the org admin. Fix: the org-level audit log sink writes to a Cloud Storage bucket in the seed project with a **locked retention policy (Bucket Lock, 400 days)**. Once locked, nobody, including the org admin, can shorten it or delete the logs. Admin Activity logs for IAM, KMS and Binary Authorization changes page the phone.

**5. Email account takeover unlocked the account.** Sign-in is an emailed code, so whoever controls the mailbox can sign in. The share model already keeps private data unreadable on a new device, but the plan did not say so, and a hijacker could still revoke keys or change the Teammate. Fix: a **new device gets no private data until an existing device approves it or the recovery code is entered**; every new sign-in notifies every device and the Teammate; destructive actions (revoke, rotate, Teammate change, export, delete) require Face ID on a device that already holds the share; passkeys ship as the second factor in the first update after launch (WorkOS supports them).

**6. A hosted graph database was a fourth party with a public endpoint.** The graph holds ciphertext, but a hosted vendor still sees graph shape, IP addresses and query timing, and the tier first considered had no private connectivity. Fix: **self-host Neo4j Community on a Confidential VM inside the VPC** (CMEK disk, no public IP, reachable only from the Cloud Run egress subnet), rebuilt from Postgres by the sync job whenever needed. One fewer company holds athlete data.

**7. Every contract went to Document AI, whether it needed to or not.** Digital PDFs and DOCX files carry their own text. Fix: **the phone extracts text first** (PDFKit for PDFs, on-device conversion for DOCX, Apple's Vision OCR for photos) and sends Markdown; Document AI is the fallback only for scans the device cannot read, with the athlete told when a document is being sent to Google for OCR. Fewer bytes leave the device.

**8. Cloud pickers ran through the server.** Fix: **Google Drive, OneDrive and Dropbox imports run entirely on the phone**; the app fetches the file with the one-shot token and uploads it to Leave already encrypted. The server never holds a cloud token.

### Medium

**9. Vendors learned about athletes through side channels.** Brandfetch queries revealed which brands an athlete has contracts with; crawler web searches could carry athlete specifics. Fix: Brandfetch lookups are keyed by brand only, cached globally, and never carry an athlete identifier; crawler queries are by market and sport, never by athlete; Claude web search calls run in the crawler only, with no athlete data in the prompt.

**10. Firebase Analytics sent behavioral events to Google Analytics.** Fix: **drop Firebase Analytics.** The tour and feature events the design lists go to a first-party events table (the same pipeline as `usage_events`), pseudonymous, exported to BigQuery in Leave's own project. Firebase stays for push, crashes and App Check; Crashlytics gets a scrubber so no strand, contract or email text can reach a crash report.

**11. Speech fallback could send audio without asking.** Fix: the Chirp 3 fallback is opt-in per athlete, and the design's "announce when audio leaves the device" becomes a visible indicator each time it happens. Leave speaking (text-to-speech) sends the text of a thread to Google; the athlete can turn it off, and it is off by default for private strands.

**12. Minors.** An earlier ruling allowed under-13 athletes in guardian-created accounts, which brings COPPA's verifiable parental consent regime, and the 2025 rule amendments (in force since April 2026) require separate consent for third-party disclosures. Fix: **minimum age 13** (decided 2026-09-23). The app's sign-up refuses any birth date under 13, and there is no under-13 path; the site's life-stage copy now reads 13 to 14 for middle school. Athletes aged 13 to 17 always have a parent or legal guardian as the account holder and required Teammate, and cannot share private data with AI apps. No public-record profile is built for anyone who has not signed up (no shadow profiles), and the third-party disclosure list (Google, Anthropic via Vertex, WorkOS, Stripe, Brandfetch, Resend, Apple) is shown at sign-up. The Texas SCOPE Act and the state app-store accountability acts are tracked by counsel; the design's parental-tools model (the Teammate) already matches their shape.

**13. Secrets and identifiers in places the redaction list did not cover.** Fix: no personal data in URL paths or query strings (opaque IDs, POST bodies), so load balancer and Cloud Armor logs stay clean; Cloud Trace attributes and Error Reporting are scrubbed the same way as the application logger; request logs kept 30 days; Vertex request-response logging confirmed off; Stripe and WorkOS carry the email and nothing else about the athlete.

**14. Domain and transport hygiene.** Fix: DNSSEC and registrar lock at Cloudflare, CAA records, HSTS preload on leaveyouragent.com, a TLS policy of 1.2 minimum with modern ciphers on the load balancer, certificate-transparency monitoring, SPF, DKIM and DMARC at reject (athletes are phishing targets, and sign-in is by email), and WorkOS's Magic Auth emails sent from Leave's own domain rather than a WorkOS domain.

**15. The phone itself.** Fix: the share is stored with `kSecAttrAccessibleAfterFirstUnlock` and synchronizable (so iCloud Keychain carries it and background Web runs can read it while the phone is locked after its first unlock; `WhenUnlocked` would fail most background runs, and the trade favours the athlete because the Secure Enclave device key still gates every destructive action); the Secure Enclave key requires biometrics or passcode for use; the app locks behind Face ID by default; private screens are hidden in the app switcher and block screenshots; the clipboard is cleared after a paste of private text; the recovery code is generated and shown once on device, and the server holds only a blob of the share encrypted under an Argon2id-derived key from that code.

### What we say to athletes, and what backs it up

- **A public security page** describing this design in plain words: what is encrypted, who holds which key, what the server can and cannot read, which vendors see what, how to revoke. **The design documents are published alongside it** (this file, `api-contract.md` and `mcp.md`), so the page's claims can be checked against the actual architecture.
- **Open-source client cryptography**: the iOS module that generates keys, wraps and unwraps data keys, and talks to the API is published, so the claim "the share never leaves the phone" can be checked.
- **An independent penetration test once Leave is funded**, then annually, with findings and fixes summarized on the security page. Until then the external checks are the disclosure policy and security.txt, the open-source client crypto module, and these public design documents. SOC 2 when a customer or investor asks.
- **security.txt and a disclosure policy** on day one; a bug bounty once the first pen test is closed.
- **A subprocessor list and a transparency report** (requests from governments or others, and Leave's answer) published twice a year.
- **Data promises with dates**: export in a standard format within a day, deletion complete within 30 days including backups, revoke effective immediately with a 24-hour undo.

### Explicitly accepted, and disclosed

- While an athlete uses Leave, the records in use are plaintext in the API's memory and in the model's context at Google and Anthropic under zero data retention. There is no way to explain a contract without reading it. The security page says so.
- Graph shape (counts, timestamps) and account metadata are visible to Leave. Support works from this only.
- Cloud Run is not confidential hardware. If Google ships it, the sealed services move first.

## 3. Production architecture in the production project

```
iPhone app ──┐                                   ChatGPT / Claude / Gemini / other MCP-capable apps
             │ HTTPS (WorkOS token)                       │ MCP over HTTPS + OAuth 2.1 (WorkOS AuthKit at auth.leaveyouragent.com)
             ▼                                            ▼
   Global external Application Load Balancer + Cloud Armor + managed certs
             │ api.leaveyouragent.com            mcp.leaveyouragent.com
             ▼                                            ▼
   Cloud Run "api" (min 1 instance)              (same service, second host rule)
     REST + MCP resource server + Stripe and WorkOS webhooks
             │ Direct VPC egress
             ├──► Cloud SQL Postgres 17 (private IP, pgvector, IAM auth)
             ├──► Cloud Storage files bucket (encrypted contracts, files, exports)
             ├──► Cloud Tasks "contracts"  ──► Cloud Run "worker" (min 0): phone-extracted Markdown
             │                                  (Document AI only for unreadable scans) → Opus 5.5 structured
             │                                  output → Postgres/pgvector → Neo4j → Brandfetch
             ├──► Secret Manager, Cloud KMS
             ├──► Vertex AI: Claude Opus 5.5 (global endpoint, zero data retention) and gemini-embedding-001
             ├──► Document AI Layout Parser (us)
             ├──► Neo4j Community on a Confidential VM in the VPC (private IP only)
             ├──► Speech-to-Text (opt-in fallback) / Text-to-Speech (Chirp 3)
             ├──► WorkOS AuthKit (identity, Magic Auth, OAuth 2.1 for MCP; webhooks back for user events)
             └──► Firebase: FCM, App Check, Crashlytics (scrubbed)

   Cloud Scheduler ──► Cloud Run Jobs "crawler" (nightly public signals per market), "reminders" (payment / deliverable dates)
   Cloud Monitoring: uptime checks on both hosts, alert policies → email + SMS
```

Cloud Run settings that matter for production: `api` with CPU always allocated, min-instances 1 and startup CPU boost (no cold start on the first "Hey Leave" of the day); `worker` request-based with min 0 and concurrency 1 (each contract gets a full CPU); gradual traffic rollout on deploy; Direct VPC egress; Cloud SQL over private IP with IAM database authentication and no password.

Cloud SQL: Enterprise edition, PostgreSQL 17 with the `vector` extension, private IP only, automated daily backups with 7-day point-in-time recovery, deletion protection on. Starts zonal on a small instance and moves to regional high availability as the athlete count grows. HNSW indexes on the embedding columns live in RAM, so memory is the first thing to grow.

Neo4j: Community edition on one Confidential VM (AMD SEV, memory encrypted in use), CMEK persistent disk, no external IP, firewall allowing only the Cloud Run egress subnet on 7687, OS Login, daily disk snapshots. Because the graph is a projection, availability is handled by the rebuild job rather than clustering; if Neo4j is down the Web screen serves the last cached view.

Cloud Storage: one files bucket, uniform access, object versioning + 7-day soft delete (a mistaken delete is recoverable, a requested delete still fully clears after the window), lifecycle rule to move old exports to colder storage after 30 days. Objects keyed `athletes/{id}/contracts/...` and `athletes/{id}/files/...` with opaque IDs; the app reaches everything through short-lived signed URLs.

## 3b. Node.js stack and practices

Decided: the backend is Node.js. Everything below is what Google's own Cloud Run guidance and current practice point to for a Node service in 2026, applied to the three workloads (`api`, `worker`, `crawler`).

### Versions and layout

- **Node 24** (Active LTS). Node 26 becomes LTS in October 2026 and Node moves to one release a year from Node 27; upgrade in the first quiet week after 26 is promoted, not before launch.
- **TypeScript, strict, ESM**, one pnpm workspace: `packages/api`, `packages/worker`, `packages/crawler`, `packages/shared` (Drizzle schema, Zod schemas, Claude prompts and tool definitions, Neo4j queries), `infra/` (Terraform). Same lockfile, same lint, one CI.
- **Fastify** for `api`. It is the Node-native choice with schema validation, a pino logger built in, first-party plugins for multipart uploads, rate limiting and security headers, and an official middleware package in the MCP TypeScript SDK.
- **MCP** through the official TypeScript SDK v2 (`@modelcontextprotocol/server` + `@modelcontextprotocol/fastify`, stateless) mounted at the root of `mcp.leaveyouragent.com`, with WorkOS AuthKit as the authorization server. See `mcp.md`.
- **Drizzle ORM** on `pg` for Postgres: SQL-close, no generation step, pgvector `vector` columns and HNSW indexes are first-class, migrations with drizzle-kit run as a Cloud Run Job before each deploy. Cold starts are noticeably faster than Prisma, which matters on `worker` (min 0).
- **Cloud SQL Node.js Connector** with `authType: 'IAM'`: no password, TLS 1.3, the service account is the database user.
- **`neo4j-driver`**: one driver per process, a session per request, parameters only (never string-built Cypher), read and write sessions split so Web screen reads never queue behind the sync job.
- **Claude on Vertex** with `@anthropic-ai/vertex-sdk` (`AnthropicVertex({ projectId, region: 'global' })`), model `claude-opus-5-5`. Three things to know about Opus 5.5: thinking is always on, and `output_config.effort` (default `medium`) is set explicitly per route: `low` for receipts and short replies, `high` for contract extraction and Web connections; forced `tool_choice` returns a 400, so extraction uses structured outputs rather than a forced tool; and it runs broader safety classifiers, so every response's `stop_reason` is checked for `refusal` before reading content. Streaming for Talk to Leave; `messages.parse()` with `output_config.format` built from the same Zod schema that types the contract row, so extraction is validated before it touches Postgres. The zero-data-retention exception turns off Vertex's default prompt cache.
- **Document AI** via `@google-cloud/documentai`, **embeddings** via the Vertex AI SDK, **WorkOS** via `@workos-inc/node`, tokens verified with `jose` against the AuthKit JWKS.
- **Zod at every boundary**: request bodies, environment variables at startup (fail fast on a missing secret), Claude structured outputs, webhook payloads from Stripe and WorkOS.

### Containers

- Multi-stage Dockerfile: build on `node:24-slim`, run `api` and `crawler` on `gcr.io/distroless/nodejs24-debian12:nonroot` (no shell, non-root, small). `worker` needs LibreOffice and libheif for DOC/DOCX and HEIC, so it runs on `node:24-slim` with those packages and a non-root user.
- `pnpm install --frozen-lockfile --ignore-scripts` in CI; `pnpm prune --prod` before the runtime stage.
- Listen on `process.env.PORT`, bind `0.0.0.0`, one process per container (Cloud Run scales instances, not workers).
- Startup: enable startup CPU boost, initialise the Postgres pool, Neo4j driver and Vertex client lazily on first use, keep `node_modules` lean, and set `--max-old-space-size` to about 75% of the instance memory.
- Graceful shutdown: on SIGTERM stop accepting connections, let in-flight requests finish, close the pool and the driver, then exit. Force `process.exit(1)` at 8 seconds because Cloud Run sends SIGKILL at 10. Never `process.exit(0)` inside the handler before cleanup finishes.
- No fire-and-forget work in `api`. Cloud Run throttles CPU outside requests on request-based services; anything that must finish goes to Cloud Tasks or the worker.
- Concurrency: `api` 80 per instance (I/O bound), `worker` 1 (a contract gets the whole CPU), `crawler` is a Job.

### Logging, tracing, errors

- **pino** to stdout as JSON with Cloud Logging's field names (`severity`, `message`, `logging.googleapis.com/trace`) so logs land structured and correlate with traces. One request ID per request, propagated to Cloud Tasks and the worker.
- **Redaction in the logger config**, not by discipline: the paths for transcripts, strand text, contract text, tokens and emails are in pino's `redact` list, which is the code-level half of the "never logged" promise.
- **OpenTelemetry** Node SDK exporting to Cloud Trace; Fastify, `pg`, and `undici` instrumented. Uncaught exceptions logged with a stack trace so Cloud Error Reporting groups them.

### Security

- Dependencies: committed lockfile, Renovate for weekly bumps, `pnpm audit` and Socket or Snyk in CI blocking high and critical, `--ignore-scripts` on install, provenance-verified publishes only. This is the supply-chain layer the OWASP npm cheat sheet describes.
- `@fastify/helmet` for headers, `@fastify/rate-limit` with the Postgres store for the per-user limits in section 2b, body limits per route (25 MB only on uploads), multipart streamed straight to Cloud Storage with a resumable upload rather than buffered in memory.
- Secrets arrive as Secret Manager mounts, read once at startup into the Zod-validated config object. Nothing from `process.env` is read elsewhere.
- Run as non-root, read-only filesystem except `/tmp`, no `eval`, no dynamic `require`.

### Testing and delivery

- **Vitest** with Testcontainers for Postgres (with pgvector) and Neo4j, so the sync job and the HNSW queries are tested against real engines. Contract fixtures come from the pitch deck's mock data (Smoothie Spot, Northline, Elite Camp Series), never real athletes.
- Recorded Claude and Document AI responses for unit tests; one live smoke test against Vertex in CI on `main` only.
- GitHub Actions with Workload Identity Federation: typecheck, lint (Biome), test, build, sign, push to Artifact Registry, run the migration Job, deploy with 10% traffic, promote after the uptime check passes.

## 4. Build order

Status markers: **done** means configured or live today; everything else is design.

**Phase 0, identity**
1. The organization already exists from the Workspace account. Confirm the founder account holds Organization Administrator, and note the Workspace customer ID for the domain-restricted-sharing policy.
2. Create the billing account.
3. Create the break-glass admin as a Workspace user, enroll security keys, enforce 2-Step Verification in the Workspace admin console.

**Phase 1, foundation (all in Terraform from here; written and validated, not yet applied)**
4. Apply the organization policies in section 2.
5. Folders `bootstrap`, `prod`, `nonprod`; the seed project with the state bucket (versioned, US multi-region), Artifact Registry, and the Workload Identity pool + provider for the GitHub repo.
6. Budget alerts on the billing account; essential contacts; Security Command Center Standard; an org-level audit-log sink into the seed project (locked, see step 14a).

**Phase 2, production project**
7. Production project: enable APIs, one VPC with a private subnet, Private Service Access range for Cloud SQL.
8. Cloud SQL instance, `leave` database, IAM database users for `api` and `worker`, `vector` extension with HNSW indexes.
8a. Document AI: enable the API, create a Layout Parser processor in the `us` location, grant `worker` the Document AI API User role.
8b. Neo4j Community on a Confidential VM in the private subnet (CMEK disk, no external IP, OS Login, snapshot schedule); store the URI and password in Secret Manager; add the outbox table and the sync job that projects Postgres changes into the graph.
9. Buckets, Secret Manager secrets (empty placeholders until phase 3), KMS key ring with the HSM key, CMEK bindings on Cloud SQL, the bucket, the Neo4j disk and Artifact Registry, and the IAM deny policy that keeps every human principal away from decrypt.
9a. Request the Vertex AI abuse-monitoring exception (zero data retention) for the production project and enable Access Transparency and Access Approval. Both take days, so file them in week one.
10. Cloud Run `api` and `worker`, Cloud Tasks queue, Cloud Run Jobs `crawler` and `reminders`, Cloud Scheduler triggers.
11. Load balancer, Cloud Armor policy, managed certificates, the two DNS records at Cloudflare.
12. Firebase on the production project: FCM with the APNs key, App Check, Crashlytics with a PII scrubber. No Firebase Auth, no Firebase Analytics.
12a. WorkOS: production environment, Magic Auth on, custom domain auth.leaveyouragent.com including the custom email domain for the code emails, branded hosted sign-in page, AuthKit for MCP with `https://mcp.leaveyouragent.com` as the resource, CIMD and DCR on (**done**); store the API key and client secret in Secret Manager and point the user-events webhook at `api` (design).
13. Monitoring: uptime checks on `api` and `mcp`, alerts for 5xx rate, p95 latency, Cloud SQL CPU and storage, Cloud Tasks backlog, usage thresholds.
14. GitHub Actions: build → cosign sign with SLSA provenance → push to Artifact Registry → Binary Authorization admits only that signature → deploy with 10% canary → promote.
14a. Org audit-log sink into a Bucket Lock bucket in the seed project (400-day locked retention); alert policies on IAM, KMS and Binary Authorization changes.
14b. Domain hygiene at Cloudflare: DNSSEC, registrar lock, CAA, HSTS preload, SPF, DKIM, DMARC at reject; load balancer TLS policy 1.2 minimum; certificate-transparency monitoring.

**Phase 3, third parties (in parallel with phase 2)**
15. Vertex AI: enable Claude Opus 5.5 in Model Garden, grant `api` and `worker` the Vertex AI User role, test structured output and gemini-embedding-001 from `worker` on the global endpoint.
16. Brandfetch, Resend (sending domain already verified for the marketing site; WorkOS sends the Magic Auth email itself, Resend stays for welcome and product email from chris@leaveyouragent.com), Stripe: the Base plan, credit top-ups and auto-refill, and the webhook endpoint. Apple Developer Program and App Store Connect (external-link entitlement for web billing), Google Play console.
16a. The `usage_events` table, the allowance checks in the API gateway, the daily and monthly caps, the kill-switch flag, and the usage screen in the app.

**Phase 4, go-live checklist**
17. Restore a Cloud SQL backup into a scratch instance and delete it (proves the backups work).
18. Load test the upload path at 20 concurrent contracts; confirm the 41-second target (phone extraction plus one Opus 5.5 pass, with Document AI only for scans) and that Cloud Tasks drains.
18a. Drop and rebuild the Neo4j graph from Postgres through the sync job, proving the graph is a projection and not a second source of truth.
19. Walk the delete path: delete a contract, confirm the object, the explanation rows, the strand edges and the KMS-wrapped text are gone.
20. Confirm Data Access logs show nothing from strands or transcripts.
21. Alerts route to a phone, not just email.
21a. Trust package live before the App Store listing: public security page, the public design documents in the leave-security repo, the open-sourced client crypto module, security.txt and the disclosure policy, the subprocessor list, and the 13+ age gate.
22. Caps drill: replay a day of recorded traffic at 5× volume against the caps and confirm the per-account daily ceiling and the global usage alert both trigger; confirm an athlete at 100% of talks gets the soft stop and the Add credits link, and that a credit purchase on the web unlocks the next turn within a minute.
23. Encryption drill: with a database dump, a bucket export, the KMS key, and a memory dump of a running API instance taken while no athlete session is active, confirm no private field decrypts. Then revoke a test athlete's key and confirm the API returns nothing readable for that account.

The independent penetration test is not on this checklist: it happens once Leave is funded, then annually (section 2d).

## Changelog

**2026-09-22**
- leaveyouragent.com is on Google Workspace, so the Google Cloud organization already exists.
- Claude runs on Vertex AI (global endpoint). No Anthropic API key in the stack.
- The talent database is a Postgres table inside Leave's own Cloud SQL database. No external data provider.
- The MCP sign-in uses the same emailed code as the app.
- Embeddings are stored in pgvector on Cloud SQL.
- The knowledge graph runs on Neo4j, projected from Postgres.
- Identity is WorkOS AuthKit: Magic Auth for the emailed code, and AuthKit as the OAuth 2.1 authorization server for MCP. Firebase Auth is not used.
- The backend is Node.js (Node 24 LTS, TypeScript, Fastify, Drizzle, pnpm workspace).

**2026-09-23**
- Private data is encrypted so only the athlete can read it: per-record data keys wrapped by both a Cloud KMS HSM key and an athlete key share held on the phone and in iCloud Keychain, with a revoke control in the app (section 2c).
- Every model call runs on Claude Opus 5.5 on Vertex AI with zero data retention; Fable 5.1 is not used because it carries a mandatory 30-day retention.
- Contracts: the phone extracts the text first; Document AI is used only for scans the phone cannot read, with the athlete told. This replaces the earlier plan to send every contract to Document AI.
- Security review applied (section 2d): the athlete share never leaves the phone; AI apps get private data only by explicit per-client opt-in; Binary Authorization with signed images; locked audit logs; new-device approval; Neo4j self-hosted on a Confidential VM in the VPC (replacing a hosted graph database); on-device cloud imports; Firebase Analytics dropped; trust package published before launch.
- Minimum age is 13, with a parent or legal guardian as account holder for athletes aged 13 to 17. There is no under-13 path.
- The talent table holds signed-up athletes only; the public record is looked up at sign-up, and no profile is pre-built for anyone who is not a user.
- Usage is metered and capped: the Base plan includes allowances, extra usage is sold as Leave credits on the web, and the gateway enforces daily and monthly caps with a kill switch. Credits are forfeited 30 days after the subscription ends unless the law requires a refund.
- API contract v1.1 written (`api-contract.md`) after the mobile app review: auth fronted by the API, device approvals, KMS-sealed stateless grants, record-key routes, Teammate wrapping via a share-derived account key pair, uploads, Web, Talk, usage, MCP clients, events and push schemas.
- MCP plan written (`mcp.md`) and reconciled: connecting happens on the web and grants public data only; private access is switched on in the app; no custom OAuth scope (`openid offline_access`); MCP served at the root path; spec 2026-07-28 with SDK v2; 5-minute access tokens; unlocks of 15 minutes or 1 hour held in memory only; athletes aged 13 to 17 cannot share private data with AI apps; the Claude directory listing waits for funding and Claude users add Leave as a custom connector until then.
- The independent penetration test happens once Leave is funded, then annually.

## Sources

- Landing zone design: https://docs.cloud.google.com/architecture/landing-zones and https://docs.cloud.google.com/architecture/landing-zones/decide-security
- Organization setup and standalone organizations: https://docs.cloud.google.com/resource-manager/docs/creating-managing-organization, https://docs.cloud.google.com/resource-manager/docs/standalone-organization-overview
- Organization policy constraints: https://docs.cloud.google.com/organization-policy/reference/org-policy-constraints
- Cloud Foundation Fabric FAST (Terraform landing-zone toolkit): https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/blob/master/fast/README.md
- Budgets and alerts: https://docs.cloud.google.com/billing/docs/how-to/budgets
- Cloud Run minimum instances: https://docs.cloud.google.com/run/docs/configuring/min-instances
- Cloud Run to Cloud SQL over private IP: https://docs.cloud.google.com/sql/docs/postgres/connect-run
- Cloud Tasks vs Pub/Sub: https://docs.cloud.google.com/pubsub/docs/choosing-pubsub-or-cloud-tasks
- WorkOS MCP, iOS SDK and Radar: https://workos.com/docs/authkit/mcp, https://workos.com/docs/sdks/ios, https://workos.com/docs/authkit/radar
- Remote MCP OAuth requirements: https://blog.modelcontextprotocol.io/posts/client_registration/, https://medium.com/@yagmur.sahin/remote-mcp-in-the-real-world-oauth-2-1-9d149de6e475
- Cloud KMS envelope encryption and CMEK: https://docs.cloud.google.com/kms/docs/envelope-encryption, https://docs.cloud.google.com/kms/docs/cmek
- Crypto-shredding on Google Cloud: https://oneuptime.com/blog/post/2026-02-17-how-to-set-up-crypto-shredding-for-gdpr-right-to-erasure-compliance-in-google-cloud/view
- Tink AEAD: https://developers.google.com/tink/aead
- Vertex AI abuse monitoring and zero data retention: https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/abuse-monitoring, https://cloud.google.com/vertex-ai/generative-ai/docs/vertex-ai-zero-data-retention
- Anthropic API data retention and Covered Models: https://platform.claude.com/docs/en/manage-claude/api-and-data-retention, https://support.claude.com/en/articles/15425996-data-retention-practices-for-covered-models
- Claude Opus 5.5 announcement and Vertex listing: https://www.anthropic.com/claude-opus-5-5, https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/claude/opus-5-5
- Access Transparency and Access Approval: https://docs.cloud.google.com/assured-workloads/access-transparency/docs/overview, https://docs.cloud.google.com/assured-workloads/access-approval/docs/overview
- Cloud EKM and Key Access Justifications: https://docs.cloud.google.com/kms/docs/ekm, https://docs.cloud.google.com/assured-workloads/key-access-justifications/docs/overview
- Confidential Space: https://docs.cloud.google.com/confidential-computing/confidential-space/docs/confidential-space-overview
- Apple passkeys, iCloud Keychain and Secure Enclave: https://support.apple.com/en-us/102195, https://support.apple.com/en-us/102651
- Passkey PRF for end-to-end encryption: https://www.corbado.com/blog/passkeys-prf-webauthn
- Binary Authorization and Sigstore: https://cloud.google.com/binary-authorization/docs/cv-sigstore-check, https://docs.sigstore.dev/cosign/signing/signing_with_containers/
- Bucket Lock for immutable logs: https://docs.cloud.google.com/storage/docs/bucket-lock
- OWASP MASVS and iOS storage/crypto tests: https://mas.owasp.org/MASVS/, https://mas.owasp.org/MASTG/0x06d-Testing-Data-Storage/, https://mas.owasp.org/MASTG/0x06e-Testing-Cryptography/
- iOS Keychain and Secure Enclave: https://www.atelier-socle.com/en/articles/keychain-security-framework-guide
- COPPA 2025 amendments and state minors' laws: https://www.ftc.gov/business-guidance/resources/complying-coppa-frequently-asked-questions, https://www.finnegan.com/en/insights/articles/coppas-amended-rule-is-now-in-full-effect-what-operators-need-to-know.html, https://www.insideprivacy.com/childrens-privacy/state-and-federal-developments-in-minors-privacy-in-2026/, https://www.privo.com/blog/what-is-the-texas-scope-act-hb-18
- ChatGPT apps and connectors data handling: https://help.openai.com/en/articles/11487775-connectors-in-chatgpt
- Node.js releases and schedule: https://nodejs.org/en/about/previous-releases, https://nodejs.org/en/blog/announcements/evolving-the-nodejs-release-schedule
- Cloud Run graceful shutdown: https://cloud.google.com/blog/topics/developers-practitioners/graceful-shutdowns-cloud-run-deep-dive
- Framework comparison: https://betterstack.com/community/guides/scaling-nodejs/fastify-vs-express-vs-hono/, https://encore.dev/articles/nestjs-vs-fastify-vs-hono
- MCP TypeScript SDK server guide: https://ts.sdk.modelcontextprotocol.io/v2/documents/Documents.Server_Guide.html
- Drizzle vs Prisma: https://www.bytebase.com/blog/drizzle-vs-prisma/, https://encore.dev/articles/drizzle-vs-prisma
- Cloud SQL Node.js Connector: https://github.com/GoogleCloudPlatform/cloud-sql-nodejs-connector, https://docs.cloud.google.com/sql/docs/postgres/connect-connectors
- Node.js security: https://nodejs.org/learn/getting-started/security-best-practices, https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html
- pino in production: https://betterstack.com/community/guides/logging/how-to-install-setup-and-use-pino-to-log-node-js-applications/
- Zod request validation: https://1xapi.com/blog/validate-api-requests-zod-nodejs-2026
