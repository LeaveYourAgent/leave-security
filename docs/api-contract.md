# Leave API contract, v1.3

Design, partly built — status as of 2026-09-25.

**What exists today**

- **Live:** the marketing site, the Terms and the Privacy Policy at leaveyouragent.com.
- **Configured:** the WorkOS production environment (custom domain auth.leaveyouragent.com verified, client ID metadata documents and dynamic client registration on). Section 10's token settings were checked against it on 2026-09-23.
- **Written, not applied:** the Terraform for the three stages, validated but not yet applied.
- **In progress:** the Node backend workspace that will serve these routes is being scaffolded; the iOS app plan is written and the app is in development.
- **Design:** the MCP server (see `mcp.md`), and every route in this document.

v1.1 (2026-09-23) applies the mobile app review's eight items: full birth date, terms consent, athlete invite from a guardian account, account deletion, encrypted export, unapproved-device scope, contract AAD continuity, and rejection-sampled account key derivation.

Companion to `architecture-and-security.md` (the infrastructure and the encryption design) and `mcp.md` (Leave inside ChatGPT, Claude, Gemini and other MCP-capable apps). Defines the routes the app calls, their shapes, and the cryptographic framing shared between the open-source client crypto module (`leave-crypto-ios`) and the Node API. Public by design.

Conventions: base `https://api.leaveyouragent.com/v1`. JSON bodies. `Authorization: Bearer <WorkOS access JWT>` on everything except `/auth/*`. Every metered response carries a `usage` object (section 9). Opaque IDs are ULIDs; no personal data ever appears in a path or query string. Binary values are base64url without padding. Errors: `{ "error": { "code": "string", "message": "plain English for the athlete", "retryAfter": seconds? } }`.

## 1. Cryptographic framing

| Item | Value |
|---|---|
| Record encryption | AES-256-GCM, 12-byte random nonce, 16-byte tag. Wire format `nonce ‖ ciphertext ‖ tag`. No Tink prefix bytes (the server configures Tink with the RAW output prefix, or uses Node `crypto` directly). |
| AAD for a record | UTF-8 of `leave:v1:<recordType>:<recordId>` where recordType is `strand`, `contract`, `contractField`, `file`, `embedding`. |
| Data key (DEK) | 32 random bytes per record, generated on the device that creates the record. |
| Athlete share | 32 random bytes, Keychain, `kSecAttrAccessibleAfterFirstUnlock`, `kSecAttrSynchronizable = true`, no biometric ACL (so it syncs). |
| Share wrap of a DEK | AES-256-GCM with key `HKDF-SHA256(share, salt = recordId, info = "leave:v1:dek-wrap")`, AAD as above. |
| Server wrap of a DEK | Cloud KMS `encrypt` on the HSM key, AAD as above, done by the API when the record is written. Both wraps are stored; both are needed to read. |
| Account key pair | P-256, derived deterministically by rejection sampling: `candidate = HKDF-SHA256(share, salt = "leave:v1:account-key", info = accountId ‖ ":" ‖ counter)` with counter starting at 0 as ASCII; accept the 32-byte output as the private scalar if it is in [1, n-1], otherwise increment the counter and retry. No modular reduction, so the phone needs no big-integer arithmetic. The test vectors pin this. Stable across the account's devices because the share syncs. Public key registered at `PUT /account/public-key`. Used to wrap DEKs to another account (the Teammate) with HPKE (DHKEM P-256, HKDF-SHA256, AES-256-GCM), info `leave:v1:xwrap:<recordId>`. |
| Device key | P-256 in the Secure Enclave, `biometryCurrentSet` or `devicePasscode` access control, never exported. Signs device approvals and destructive-action confirmations (section 4). Not a wrapping key. |
| Session sealing | The server publishes an RSA-3072-OAEP-SHA256 public key held in Cloud KMS (HSM). The app generates a 32-byte session key, seals it with that public key, and seals each per-record DEK it wants the server to use with AES-256-GCM under the session key, AAD `leave:v1:grant:<sessionId>:<recordId>`. The server unseals the session key through KMS `asymmetricDecrypt` (one KMS call per grant, cached in instance memory until expiry) and the DEKs in memory. Nothing is persisted. Because the private key lives in KMS, any Cloud Run instance can unseal; no session affinity is needed, and no human principal holds decrypt rights. |
| Grant | `{ sessionId, sealedSessionKey, exp (unix seconds, at most now+900), records: [{ id, type, sealedDek }] }` sent as the request body of `POST /session/keys` and echoed by the app in the `X-Leave-Grant` header (base64url JSON) on each request that needs it. Stateless: the server never has to remember a grant. (MCP unlock grants in section 10 are the one exception: they are held in API memory for their lifetime.) |
| Recovery blob | `version(1 byte = 0x01) ‖ salt(16) ‖ nonce(12) ‖ AES-256-GCM(share) ‖ tag(16)`, key `= Argon2id(recoveryCode, salt, memory 64 MiB, iterations 3, parallelism 1, output 32)`. Recovery code: 28 characters from a 32-symbol Crockford base32 alphabet (140 bits), shown once on device. The server stores the blob only. |
| Test vectors | `leave-crypto-ios/TestVectors/v1.json` is the shared source; the server's `packages/shared/crypto/vectors.test.ts` loads the same file. |

## 2. Auth (WorkOS behind Leave routes, designed screens kept)

`POST /auth/code` `{ email }` → `202 {}` always (no account enumeration). The server calls WorkOS Magic Auth with its API key; the email comes from Leave's own domain via auth.leaveyouragent.com. Rate limits per section 2b of `architecture-and-security.md`.

`POST /auth/verify` `{ email, code, device?: { name, model, publicKey (SE P-256, SPKI), attestation (App Attest) } }` →
`200 { accessToken, refreshToken, accountId, isNewUser, deviceId, deviceApproved, approvalRequestId? }`
`deviceApproved` is true for the first device on a new account and false for any later device until section 4 completes. Errors: `invalid_code` (400; five wrong attempts invalidate the code), `code_expired` (400, after 10 minutes), `rate_limited` (429 with `retryAfter`). `device` may be omitted (v1.3): the token is then device-less, `deviceId` is null, and it may call only the device-less routes listed in section 4; the first device registered on an account with no approved device is approved automatically. No private data can exist on an account until it has an approved device.

`POST /auth/refresh` `{ refreshToken }` → `200 { accessToken, refreshToken }`. Access tokens 15 min, refresh 30 days, rotated on use.

`POST /auth/logout` `{ refreshToken }` → `204`.

## 3. Account and Teammate

`POST /account/profile` `{ role: "athlete" | "guardian", firstName, lastName, birthDate: "YYYY-MM-DD", athleteFirstName?, termsVersion }` → `201 { accountId }`. Called once after the first `/auth/verify` with `isNewUser`. The server computes age from the full date; under 13 → `error.code = "age_minimum"` and no account is created. `termsVersion` records acceptance of the Terms and Privacy Policy with a server timestamp (no device signature; the device may not exist yet).

`GET /account` → `{ accountId, email, role: "athlete" | "guardian" | "teammate", firstName, lastName, birthDate, teammate?: { accountId, name, status: "empty"|"pending"|"active", publicKey, required: bool }, guardian?: { accountId, name }, athletes?: [{ accountId, firstName, status: "invited"|"active" }], termsAccepted: { version, at }, publicKey }`

`PUT /account/public-key` `{ publicKey }` (account P-256, SPKI) → `204`. Set on first device, unchanged unless the share is rotated.

`POST /consent` `{ scope: "terms" | "third_party_disclosure", version?, method?, evidence? }` → `204`. `terms` needs no device signature; `third_party_disclosure` is device-signed by the parent or guardian.

`POST /athlete/invite` `{ email, athleteFirstName }` on a guardian account → `202`, device-signed. The athlete (aged 13 to 17) signs in with the same emailed code, completes `POST /account/profile` as role `athlete`, generates their own share on their first device, and the parent or guardian is linked as the required Teammate on the athlete's account (`teammate.required = true`, cannot be removed by the athlete). The guardian's phone never holds the athlete's strand keys; it receives only contract keys wrapped to the guardian's account public key, like any Teammate.

`DELETE /athletes/{id}` on a guardian account, device-signed → `202 { completesBy }`. The Privacy Policy promises the parent or guardian can delete an athlete's account; this runs the athlete's revoke and deletion (section 5), notifies the athlete's devices, and unlinks the guardian. The guardian never receives the athlete's private keys in the process.

`POST /teammate/invite` `{ email }` → `202`. Requires a device-signed confirmation (section 4).
`GET /teammate/key` → `{ accountId, publicKey }` for wrapping contracts.
`PUT /contracts/{id}/teammate-key` `{ hpkeWrappedDek }` → `204`.
`DELETE /teammate` → `204`, device-signed; the server deletes every Teammate wrap. For an athlete aged 13 to 17 the parent or guardian is the Teammate and cannot be removed.

Teammate change of phone: the Teammate's share syncs through their own iCloud Keychain, so the account key pair is the same on the new phone and existing wraps keep working. Teammate replaced: the athlete's app re-wraps every contract DEK to the new Teammate's public key and calls `PUT /contracts/{id}/teammate-key` for each; the old wraps are gone at `DELETE /teammate`.

## 4. Devices and approvals

`POST /devices` `{ name, model, publicKey, attestation }` → `{ deviceId, approved: false, approvalRequestId }` (also done implicitly by `/auth/verify`).
`GET /devices` → `[{ deviceId, name, model, createdAt, lastSeenAt, approved, isCurrent }]`.
`POST /devices/{id}/approve` `{ approvalRequestId, wrappedShare, signature }` → `204`. An existing approved device wraps the share to the new device with HPKE to the new device's SE public key (info `leave:v1:device-approve:<approvalRequestId>`) and signs `approvalRequestId ‖ newDevicePublicKey` with its own SE key. The server verifies the signature against a known approved device and stores the wrapped share for one-time pickup.
`GET /devices/pending` → `[{ approvalRequestId, deviceId, name, model, requestedAt, wrappedShare? }]` (the new device polls until `wrappedShare` appears, or the recovery code path is used instead).
`DELETE /devices/{id}` → `204`, device-signed.

Unapproved or device-less scope: an access token from `/auth/verify` with `deviceApproved: false` (or no device) may call only `GET /account`, `POST /account/profile`, `POST /consent`, `GET /devices`, `POST /devices`, `GET /devices/pending`, `GET /recovery`, `POST /devices/{id}/activate`, `POST /auth/refresh`, `POST /auth/logout`, `GET /usage`, `POST /events`, and `DELETE /account` while the account has no approved device (see section 5). Every other route returns `403` with `error.code = "device_unapproved"`, so the app renders the approval screen from that code. After the recovery-code path, the device proves possession of the share with `POST /devices/{id}/activate` `{ proof }` where proof is a signature by the account key pair over `deviceId ‖ accountId`; the server verifies against the registered account public key and marks the device approved.

Destructive-action confirmation: the app signs `action ‖ targetId ‖ timestamp` with the SE key (biometric prompt) and sends `X-Leave-Confirm: <base64url JSON { deviceId, timestamp, signature }>`. Required on: teammate invite/remove, key rotate, key revoke, export, delete account, device removal, turning on private data for an MCP client, guardian consent changes. Timestamp within 5 minutes.

## 5. Recovery and key lifecycle

`PUT /recovery` `{ blob }` → `204` (replace). `GET /recovery` → `{ blob }` (only after `/auth/verify`; rate limited 5 per hour per account; each fetch notifies all devices).
`POST /key/rotate` `{ newPublicKey, rewrapped: [{ recordId, shareWrappedDek }] , newRecoveryBlob }` device-signed → `204`. Paged if over 1,000 records: `POST /key/rotate/begin` → `{ rotationId }`, `PUT /key/rotate/{rotationId}/records` in batches, `POST /key/rotate/{rotationId}/commit`.
`POST /key/revoke` device-signed → `{ undoUntil }`. The server marks the account revoked; reads fail immediately; at `undoUntil` (24 h) the server deletes every share-wrapped DEK and the recovery blob. `POST /key/revoke/undo` within the window restores. The app deletes the share from Keychain and iCloud Keychain at revoke and can only undo if the athlete kept the recovery code.
`GET /key` → `{ createdAt, rotatedAt, devices: n, recoverySet: bool, revoked?: { undoUntil } }`.

`DELETE /account` device-signed → `202 { completesBy }`. While the account has no approved device (nothing encrypted exists yet), the confirmation header is not required (v1.3). (30 days: revoke runs first, then ciphertext, metadata and backups are gone by `completesBy`). A guardian account with a linked athlete is refused with `error.code = "linked_athlete"` until the athlete's account is deleted or a replacement parent or guardian has accepted, so deleting a parent's account can never destroy or orphan the athlete's data.

`POST /export` `{ dek: { shareWrappedDek } }` device-signed, 2 per day → `202 { exportId }`. `GET /export/{id}` → `{ status: "building"|"ready"|"expired", url?, expiresAt? }`. The archive (JSON plus original files) is encrypted under that DEK with AAD `leave:v1:export:<exportId>`, so support cannot read an export either. Ready within a day; the signed URL lives 7 days.

## 6. Session grants and record keys

`POST /session` → `{ sessionId, serverPublicKey (RSA-3072 SPKI), keyVersion, exp }`.
`POST /session/keys` body = grant (section 1) → `204`. The server validates the seal and caches the DEKs in memory until `exp`.
`DELETE /session` → `204` (drops the cache early).
Any request that reads private data also carries `X-Leave-Grant`; if the instance has not seen it, it unseals it (one KMS call) and proceeds.

`GET /records/keys?ids=a,b,c` → `[{ recordId, type, shareWrappedDek, serverWrappedDek, teammateWrappedDek? }]` (max 200 per call).
`PUT /records/keys` `[{ recordId, shareWrappedDek }]` → `204` (used by rotation and by re-wraps after a new record is created off-device, e.g. via MCP: the server creates the DEK, server-wraps it, and stores the share-wrap as pending; the next app session fetches pending records with `GET /records/keys?pending=true` and completes the share wrap).

## 7. Contracts, files, uploads

The phone extracts a contract's text first (PDFKit for PDFs, on-device DOCX conversion, Apple's Vision OCR for photos) and uploads Markdown alongside the encrypted original. Document AI is used only for scans the phone cannot read, and only after the athlete is told and agrees. The worker then runs Gemini 3.8 Flash structured extraction.

`POST /uploads` `{ kind: "contract"|"file", contentType, sizeBytes, sha256 }` → `{ uploadId, signedUrl, expiresAt, remainingToday }`. The client encrypts before PUT to the signed URL (record type `file` or `contract`, AAD uses `uploadId`).
`POST /contracts` `{ uploadId, markdownUploadId?, ocrByGoogle: false|true, pageCount, dek: { shareWrappedDek } }` → `{ contractId, status: "processing", etaSeconds }`. `ocrByGoogle` may be true only after the athlete agreed on screen; the server rejects a scan without Markdown unless it is true. Requires a grant containing the contract's DEK.
AAD continuity: the uploaded object is encrypted with AAD `leave:v1:contract:<uploadId>` (or `file`) before the contract exists, and keeps that AAD forever; `GET /contracts/{id}` returns `uploadId` so the app can decrypt the original. Derived `contractField` records use `leave:v1:contractField:<contractId>`.

`GET /contracts/{id}` → `{ contractId, uploadId, status: "processing"|"pendingOk"|"ready"|"failed", brand: { name, domain, colors, logoUrl }, encryptedFields (AES-GCM under the contract DEK): { term, deliverables, usageRights, paymentDates, exclusivity, termination, flags[] }, payments[], teammateWrapped: bool }`.
`POST /contracts/{id}/confirm` → `204` (Pending your OK → saved).
`DELETE /contracts/{id}` → `204`; deletes object, fields, embeddings, graph nodes and all wraps.
`GET /files`, `GET /files/{id}` → signed read URL (15 min), `DELETE /files/{id}`.
`GET /storage` → `{ usedBytes, quotaBytes }`.

Client platform notes: the app's deployment target is iOS 17 (CryptoKit HPKE); `POST /talk` is consumed with an XHR-based SSE client because React Native's fetch does not stream, so the server sends `text/event-stream` with heartbeats every 15 s.

Client-side conversion notes (from the app plan, recorded here so the server matches): DOCX is converted on the phone (unzip, read `word/document.xml`); OCR confidence under 0.6 triggers the "send to Google's OCR?" prompt; cloud pickers use `ASWebAuthenticationSession` with PKCE and keep the token in memory only.

## 8. Web, Talk, pitches

`POST /web/run` (grant with strand DEKs) → `{ runId, newConnections: n, remainingToday }`. The server decrypts strand embeddings in memory, queries the public HNSW index, asks Gemini 3.8 Flash to score and phrase candidates, and writes threads encrypted under new DEKs (share-wrap pending, section 6).
`GET /web` → `{ strands: [{ recordId, encrypted }], signals: [...public...], connections: [{ recordId, encrypted, blockedUntil?, evidence: [...public...] }] }`.
`POST /talk` (SSE; grant with the strands and contract the turn touches) `{ text?, audioTranscript?, context: { contractId? } }` → events `token`, `receipt` (encrypted change list with undo tokens), `done { usage }`.
`POST /talk/undo` `{ undoToken }` → `204`.
`POST /pitch` `{ connectionId }` (grant) → `{ pitchId, encryptedDraft, usage }`.

## 9. Usage and billing

The Base plan is $99 a month with a 30-day trial; its monthly allowances (300 talks, 5 contracts / 100 pages, 30 pitch drafts, 120 speaking minutes, 5 GB, Web runs up to 4 a day) and the hard caps are listed in section 2b of `architecture-and-security.md`. Extra usage is Leave credits, 1,000 for $10, bought on the web.

`GET /usage` → `{ periodStart, periodEnd, trial: bool, allowances: { talks: { used, included }, contractPages: {...}, contracts: {...}, pitches: {...}, speakingMinutes: {...}, webRunsToday: { used, cap } }, credits: { balance, autoRefill: { enabled, monthlyCapUsd } } }`.
Every metered response includes `usage: { talks: [used, included], credits: balance, nudge: null | "80" | "100" }`.
`GET /billing/portal` → `{ url }` (Stripe customer portal on leaveyouragent.com; the app opens it in Safari). `GET /billing/credits/checkout?pack=1000` → `{ url }`.
Accounts for athletes aged 13 to 17: billing routes require the parent or guardian's token. Credits never expire while the subscription is active; they are forfeited 30 days after it ends except where a refund is required by law (the `credits` balance is zeroed by the reminders job on day 30 after cancellation).

## 10. MCP clients

Full design in `mcp.md`.

Canonical resource: `https://mcp.leaveyouragent.com` with MCP served at the root path, so the address the athlete types is exactly the protected-resource metadata `resource` (Claude requires the match). MCP spec 2026-07-28, stateless; the server uses SDK v2 (`@modelcontextprotocol/server` + `@modelcontextprotocol/fastify`, `createMcpHandler` in stateless mode), which serves the 2025 spec too. No custom OAuth scope (WorkOS answers unknown scopes with `invalid_scope`, tested live 2026-09-23): clients request `openid offline_access`, the protected-resource metadata omits `scopes_supported`, and `WWW-Authenticate` omits `scope`. Access is decided by `aud = https://mcp.leaveyouragent.com` plus the user, and the server enforces the per-client private flag, so turning private access on never forces a re-authorization.

Connecting happens on the web only (the hosted WorkOS page inside the AI app's sign-in sheet) and grants public access only. There is no consent universal link and no `/mcp/consent` route: listing auth.leaveyouragent.com as an associated domain would let the installed app capture the OAuth page mid-flow inside the system auth sheet ChatGPT and Claude use, breaking sign-in. Private access is turned on afterwards, in the app.

`GET /mcp/clients` → `[{ clientId, name, provider: "chatgpt"|"claude"|"copilot"|"gemini"|"lechat"|"perplexity"|"grok"|"cursor"|"vscode"|"other", private: bool, unlockedUntil?, connectedAt, lastUsedAt }]`. Provider is derived from the CIMD `client_id` URL or the DCR `client_name` and `redirect_uri`. The `copilot` value exists only for Copilot Studio clients that connect on their own; Leave does not market Copilot.
`PUT /mcp/clients/{id}/scopes` `{ private: bool }` device-signed when turning private on → `204`. The app shows the provider's retention notice before enabling. Accounts aged 13 to 17 cannot share private data with an AI app: the server refuses `private: true` and refuses `POST /mcp/clients/{id}/grant` for these accounts with `error.code = "minor_private_blocked"`, and the app does not show the toggle.
`DELETE /mcp/clients/{id}` → `204` (revokes the WorkOS grant and drops any in-memory keys).
`POST /mcp/clients/{id}/grant` body = a grant (section 1) for that client's records plus `{ ttlSeconds: 900 | 3600 }` (15 minutes default, 1 hour maximum) → `204`. `DELETE /mcp/clients/{id}/grant` → `204` ("Lock now"). MCP grants are held in API process memory only, never in Postgres or a cache; a scale-out or restart means the next private read asks for an unlock again. A private read with no live grant returns a friendly "Open Leave to unlock private data for <client>" tool result and sends the `mcp_unlock_request` push; one tap in the app posts a new grant. Public tools and all writes work at any time.

MCP OAuth tokens (WorkOS AuthKit): access tokens last 5 minutes (the WorkOS production default, kept so Disconnect takes effect within 5 minutes), refresh tokens 30 days rotated on use, PKCE S256, resource indicator `https://mcp.leaveyouragent.com` checked by the server as `aud`. Issuer `https://auth.leaveyouragent.com`, JWKS at `/oauth2/jwks`. WorkOS sessions sign out after 30 days idle, so connectors do not sign athletes out every few days. Client ID metadata documents and dynamic client registration are both on (DCR is deprecated in the 2026-07-28 spec but still used by clients). Verified live on 2026-09-23: custom domain issuer, JWKS path, CIMD and DCR both on, S256, token auth `none`, loopback redirects on any port, the resource configured. Still open: `iss` in the authorization response (needs a real sign-in once the MCP server exists). The code input must render with `autocomplete="one-time-code"`.

Tools (model-free except `add_contract`, which enters the contract pipeline): `get_profile` (public record), `get_public_signals`, `list_contracts` (metadata only), `get_contract` (private, needs unlock), `get_web_threads` (public parts always; private strand text only when unlocked), `add_note` (write; becomes an encrypted strand, share-wrap pending), `add_contract` (write; enters the contract pipeline and draws from the contracts allowance), `unlock_private` (sends the push). No delete or send tools. Per-client limits: 60 calls a minute, 600 a day; every call writes a `usage_events` row with feature `mcp`.

Distribution: Claude users add Leave as a custom connector until Leave can apply for the Claude directory listing, which waits for funding.

## 11. Events and push

`POST /events` `[{ name, ts, props }]` → `202`. Pseudonymous ID: a random UUID generated on install, stored in the Keychain (not synced), sent as `X-Leave-Install`; the server never joins it to the account ID. Names from the design: `tour_started`, `tour_step_viewed`, `tour_skipped`, `tour_completed`, `first_action_picked`, plus `usage_nudge_shown`, `credits_purchased`.

Push payloads (APNs, `content-available` where noted; never private content):
- `new_sign_in`: `{ type, deviceName, model, city, at }`
- `device_approval_request`: `{ type, approvalRequestId, deviceName, model }`
- `new_public_connection`: `{ type, market, signalKind }` (text like "A new casting call in Philadelphia")
- `usage_nudge`: `{ type, allowance, percent }`
- `web_run_wanted`: `{ type }` (`content-available`, prompts a background matching run)
- `payment_due`: `{ type, contractId, dueDate }` (the brand name is shown only after decryption in the app)
- `mcp_connected`: `{ type, clientId, clientName, provider }` (refreshes Connections and doubles as a security alert: "Leave was connected to ChatGPT. Not you? Disconnect.")
- `mcp_unlock_request`: `{ type, clientId, clientName, provider }` (one tap opens the unlock sheet; the app posts a grant with the chosen duration, 15 minutes or 1 hour; never sent to accounts aged 13 to 17)

## 12. Age and consent

Roles: `athlete` (an adult who holds their own account, or an athlete aged 13 to 17 invited by a parent or guardian), `guardian` (the parent or legal guardian who holds the account for an athlete aged 13 to 17 and is the required Teammate), `teammate`. **Minimum age is 13** (decided 2026-09-23). `POST /auth/verify` for a new account is followed by `POST /account/profile` with the full `birthDate`; the server computes age and rejects under 13 with `error.code = "age_minimum"`, creating no account. For 13 to 17 the parent or guardian creates their own account, gives the third-party disclosure consent (`POST /consent`, device-signed), invites the athlete with `POST /athlete/invite`, is linked as the athlete's required Teammate, and holds billing. The athlete has their own account, share and devices, so private strands are never readable by the guardian. Athletes aged 13 to 17 cannot share private data with AI apps (section 10). There is no under-13 path.

The talent table holds records only for athletes who have signed up; an athlete's public record is looked up at sign-up, and no profile is pre-built for anyone who is not a user.
