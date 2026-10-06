# Leave in your AI chat: the MCP design

Built and running; proven from one AI app. Status as of 2026-10-06.

**What exists today**

- **Running:** the MCP server at `https://mcp.leaveyouragent.com` (the discovery document, the token check and the eight tools in section 4), the sign-in at `auth.leaveyouragent.com`, and the page `leaveyouragent.com/connect`.
- **Proven with a real account:** Claude only. A Claude custom connector signed in with the emailed code, listed the tools and read the public profile (2026-09-29 and 2026-09-30).
- **Known defect, found 2026-10-06:** a sign-in code made by the hosted sign-in page is not emailed. WorkOS's own code email is switched off in production, and Leave's server emails only the codes the Leave app asks for. This has been so since 2026-10-04 01:12 UTC, when the code email moved to Leave's own design and the hosted page was not considered. Until it is fixed, no AI app can complete a new sign-in. The fix is being built: the server sends Leave's own code email for a code the hosted page asks for.
- **Not yet proven:** ChatGPT, Gemini and every other app in section 2 have never connected to the production server. The private unlock from the phone (section 5) has not been walked end to end on a phone. Disconnect has been tested against a stand-in for WorkOS, not yet against WorkOS in production. Saving a note and adding a contract from an AI app have not been run with a real account.
- **Not submitted:** no directory or store listing has been submitted for any AI app.
- **In testing:** the Connections screen in the iOS app. Nothing app-side is called live until the app is in the App Store.
- **Accounts:** Leave is invite-only for now, so connecting needs an account that already exists.

This design builds on `architecture-and-security.md` (infrastructure and encryption) and `api-contract.md` (the routes and push payloads it refers to). It is a public document, so it contains no keys, IDs or internal hostnames.

## The goal

An athlete adds Leave to the AI they already use (ChatGPT, Claude, Gemini and other MCP-capable apps) in under a minute. They type one address, or tap one button, and sign in with the emailed code. Private data never leaves Leave unless the athlete turns it on for that AI, on their phone, with Face ID. When cost and convenience pull against the athlete's safety, the athlete wins.

## 1. What the athlete does

**Address (for any chat that asks for one): `https://mcp.leaveyouragent.com`**, with no path. It is the shortest thing we can ask anyone to type.

1. Tap **Add to ChatGPT / Claude / Gemini / …** in the app (Profile → Connections) or on **leaveyouragent.com/connect**.
2. The AI opens a Leave sign-in page at `auth.leaveyouragent.com`. The athlete enters their email and types the 6-digit code; iOS offers the code from Mail. One more screen reads "ChatGPT will see your public record and public Web threads. It can add things you tell it to Leave. Your private notes and contracts stay locked." Then they tap **Allow**.
3. That's it. Public data and writes work right away.
4. **Optional:** to share private notes and contracts, go to Connections → ChatGPT → **Share private data**. That shows the provider's retention notice and asks for Face ID, and the athlete picks how long it stays unlocked: 15 minutes (the default) or 1 hour. Athletes aged 13 to 17 don't get this option (section 5). **Lock now** ends it at any time.

**Why the private step is not part of the connect flow:** a universal link on `auth.leaveyouragent.com` would let the installed Leave app take over the OAuth page in the middle of sign-in. ChatGPT and Claude run sign-in in a system auth sheet, which can be dismissed while the athlete is away in another app. The result is a broken connect flow. Connecting therefore stays entirely on the web and is public only, and private access is switched on later inside the app. This is simpler for the athlete, more reliable, and just as strict, because private data still needs the share on the phone.

- `auth.` and `mcp.leaveyouragent.com` are **not** in the app's associated domains (the apple-app-site-association file on iOS and assetlinks on Android).
- There is no consent universal link and no `/mcp/consent` route. `PUT /mcp/clients/{id}/scopes` (device-signed) and `POST /mcp/clients/{id}/grant` cover the in-app step.
- If a universal link is ever needed, only specific paths go in the apple-app-site-association file (for example `/open/*` on the main domain), so OAuth pages always stay in the browser.
- There is no custom OAuth scope. WorkOS rejects unknown scopes with `invalid_scope` (tested 2026-09-23), so clients ask for `openid offline_access`, and access is decided by the token audience and the signed-in athlete. The server enforces the private switch for each client, so turning private data on never makes the athlete reconnect.

## 2. Per-client setup (what the athlete sees and what we must do)

Only the Claude row has been tested with a real account. Every other row is taken from that provider's own documentation and has not been tested against Leave.

| Client | How the athlete adds Leave | Auth | One-click listing | Limits to know |
|---|---|---|---|---|
| **Claude** (web, desktop, mobile) | Button → `claude.ai/customize/connectors?modal=add-custom-connector&mcpName=Leave&mcpServerUrl=https://mcp.leaveyouragent.com`. This link is **unofficial**, so fall back to "Customize → Connectors → Add custom connector → paste the address". | CIMD | Connectors Directory. The submitter must be a Claude Team or Enterprise org, and every tool needs a title and annotations. Leave applies after funding. | The Free plan allows 1 custom connector. Once listed, Leave doesn't count against that limit. Claude.ai is 18+. |
| **ChatGPT** | Listed plugin (one click in the directory). Before listing, only Developer mode works. | CIMD, or DCR as a fallback. No static client ID. | Plugin directory on the OpenAI Platform. Needs business and domain verification, tool annotations, and 5 positive plus 3 negative test cases. | Free users can only add *listed* plugins, so reaching them needs a listing. Nothing has been submitted (section 7). Publishing was not open in the EEA, UK and Switzerland at launch. |
| **Gemini** | Settings → Connected Apps → Add a custom app → paste the address | DCR | No third-party directory | US only, 18+, and Keep Activity must be on. |
| **Le Chat** | Connectors → Add custom MCP connector → paste | DCR | Built-in directory. The submission path is unclear, so we will ask Mistral. | Available on the Free plan. |
| **Perplexity** | Settings → Connectors → custom → paste | DCR | None | Pro and above. |
| **Grok** | grok.com/connectors → New → Custom → paste | Assumed DCR; test it before claiming support | Curated only | Unverified. |
| **Cursor** | `cursor://anysphere.cursor-deeplink/mcp/install?name=Leave&config=<base64 {"url":"https://mcp.leaveyouragent.com"}>` | DCR | Marketplace | None noted. |
| **VS Code** | `vscode:mcp/install?<url-encoded {"name":"leave","type":"http","url":"https://mcp.leaveyouragent.com"}>` | CIMD | MCP gallery | None noted. |
| **Claude Code** | `claude mcp add --transport http leave https://mcp.leaveyouragent.com` | CIMD (loopback) | Anthropic directory | None noted. |

Where the site names AIs, it names ChatGPT, Claude and Gemini, then "other MCP-capable apps". Leave only promises clients an athlete can actually connect.

**Copy button everywhere:** every row has a fallback card with the address, a copy button and 3 illustrated steps. This handles the moment a provider changes its menus.

## 3. Authorization (WorkOS AuthKit; our server only checks tokens)

We follow the MCP authorization spec **2026-07-28**, which is stateless and deprecates DCR but keeps it working, and we still serve clients that speak 2025-11-25.

WorkOS settings:
- Both "Client ID Metadata Document" (off by default) and "Dynamic Client Registration" are on. ChatGPT, Claude and VS Code use CIMD. Gemini, Le Chat, Perplexity, Cursor and probably Grok need DCR.
- Resource indicator `https://mcp.leaveyouragent.com`. The token `aud` must equal it exactly, with no trailing slash.
- PKCE S256 is mandatory. Refresh tokens are always issued (`offline_access`). Access tokens last 5 minutes; refresh tokens last 30 days and rotate on use; a dead refresh token returns `invalid_grant`.
- Redirect addresses come from each client's own registration (its metadata document, or dynamic registration). Loopback redirects (`localhost` and `127.0.0.1`, any port) were tested for Claude Code and VS Code.
- Branded custom domain `auth.leaveyouragent.com`. The code emails come from Leave's own domain, which is the best defence against phishing lookalikes.

Our MCP server:
- Serves `/.well-known/oauth-protected-resource`, which lists the resource and the AuthKit issuer `https://auth.leaveyouragent.com`, with no `scopes_supported` list.
- Rejects unauthenticated calls with `401` + `WWW-Authenticate: Bearer resource_metadata="…"` (no `scope` parameter).
- Verifies the WorkOS JWT against the JWKS, checking `iss`, `aud`, `exp` and the user.
- Never passes a token on to another service, and never treats any handle it returns as proof of who someone is.

**WorkOS state on 2026-09-23 (done, in production):**
- CIMD and DCR are on.
- The resource `https://mcp.leaveyouragent.com` is set.
- `auth.leaveyouragent.com` is verified, and the issuer is `https://auth.leaveyouragent.com`.
- Magic Auth is on, and passwords and passkeys are off. (Passkeys come back as a second factor in the first update after launch; see `architecture-and-security.md`, finding 5.)
- SSO is off.
- Sessions sign out after 30 days idle.
- The sign-in page carries Leave branding, with Terms and Privacy links.
- Since 2026-10-05 the sign-in and sign-up pages carry this text beside the form (under it on a narrow screen): "Leave accounts are made in the Leave app. This page signs in an account you already have." It is the same for everyone, so it says nothing about any address. Only people connecting an AI app see this page; the Leave app signs people in on its own screens.
- Since 2026-10-06 the page's "Sign up" link, and the hosted sign-up page itself, lead to `leaveyouragent.com/connect`, so no sign-up form is offered there. Sign-up in the Leave app is unchanged (checked the same minute: a new address still got its code). The sign-in form still sends a code to any address typed, which is why a sign-in with no Leave account gets the plain sentence in section 4 and is deleted after a day.
- Sign-in code emails are sent from leaveyouragent.com, verified through SPF and two DKIM records at Cloudflare.
- The metadata advertises S256 and token auth `none`, so checks 2 and 3 below pass. A test client with loopback redirects was accepted on any port, then deleted.
- Access tokens last 5 minutes (the WorkOS default). This is stricter than the 1 hour first planned, because a Disconnect takes effect within 5 minutes. Kept.

**Checks to run against WorkOS before launch:**
1. The authorization response includes `iss`. ChatGPT compares it exactly. (Open: the sign-in server does not advertise it, and no ChatGPT sign-in has been run.)
2. `token_endpoint_auth_methods_supported` includes `none`. Without it, Claude won't use CIMD. (Passes.)
3. Loopback redirects work on any port. (Passes.)
4. The hosted code field sets `autocomplete="one-time-code"`. (Open.)
5. The sign-in server advertises the device-code grant, which no supported AI app needs. (Open: WorkOS shows no switch for it, and the question to WorkOS has not been answered.)

## 4. The server

- Runs inside the existing `api` Cloud Run service, with a second host rule on the existing load balancer. Cloud Armor, Binary Authorization and the warm instance (min-instances 1) come with it, so there is no cold start and no new service to run or secure.
- MCP SDK v2 for TypeScript (`@modelcontextprotocol/server` + `@modelcontextprotocol/fastify`) with `createMcpHandler` in stateless mode. There are no session IDs and no session store; one handler serves both spec versions.
- The read tools are model-free, so they never call the model. Each call writes a `usage_events` row with `feature = "mcp"` and no private content.
- Rate limits: 60 calls a minute and 600 a day per client, plus Cloud Armor per-IP limits and a flood guard on the MCP host and the OAuth routes.
- Every tool call is recorded with the tool, the AI client and how it ended (ok, locked, private, refused or error). Arguments and results are never recorded.
- A request from another site's `Origin` is refused. A token that does not name the client it was issued to is refused.
- An account that is deleted loses its sign-in identity, so every AI connection it had stops and has to sign in again.
- A person who signs in with no usable Leave account (none, sign-up unfinished, or being deleted) gets one plain sentence from every tool, for example "There is no Leave account for this sign-in. Leave accounts are made in the Leave app." No account data is read and no connection is recorded. Before 2026-10-05 the AI app was asked to sign in again without end. Covered by the server's tests; not yet seen with a real sign-in in production.

### Tools

Every tool has a `title`, annotations, an `outputSchema` and `structuredContent`. They come back in a fixed order, and their descriptions are never changed after approval.

| Tool | Reads/writes | Annotations | Data |
|---|---|---|---|
| `get_profile` | read | readOnly | Your public record in Leave: sport, position, team or school, class and follower counts |
| `get_public_signals` | read | readOnly, openWorld | Public opportunities near you: casting calls, brand posts, sponsor programs, press. Each carries the day its page was published (or none), whether it is a standing program, and the day the crawl last found it. Only signals inside the age limit of `api-contract.md` are listed, and nothing says a program is open now |
| `list_contracts` | read | readOnly | Brand, status and dates only |
| `get_contract` | read | readOnly | Terms and flags. **Needs an unlock** |
| `get_web_threads` | read | readOnly, openWorld | Threads in your Leave Web. Public parts are always included; the private reason ("you told Leave…") only while unlocked |
| `add_note` | write | not destructive | Saves something the athlete tells the AI as an encrypted private note (strand) and creates one embedding with Google's embedding model; 100 a day at most. Needs Leave's AI to be on for the account (since 2026-10-06) |
| `add_contract` | write | not destructive | Sends contract text into the contract pipeline (Gemini 3.8 Flash extraction) under the same plan, consent, page and storage checks as an upload from the app |
| `unlock_private` | action | not read-only (it reaches the phone) | Sends the `mcp_unlock_request` push; returns no data. One push a minute and five an hour per AI client |

The server also sends `instructions` with `initialize`: the person in the chat is addressed as "you" and by the first name on their Leave account (a guardian is named as the parent or guardian on the athlete's account), the AI calls `get_profile` first when greeted, and Leave gives information rather than instructions: what was published and when, and what is due or asked for. The instructions say that nothing in Leave says a program is open now. For an athlete under 18, and for a parent or guardian's connection, they also tell the AI to name no brand in the eight categories that are for athletes 18 and older (2026-10-05).

Both write tools need the account's AI consent. Without it they answer `ai_consent_required` with one sentence: "Leave's AI is off on this account, so nothing was saved or sent to Google. It is turned on in the Leave app, under Profile › AI and Google." For an athlete of 13 to 17, or a parent or guardian's connection, the sentence says a parent or guardian turns it on from their own Leave app. A refused note saves nothing, embeds nothing and uses none of the day's 100. No note was added through an AI connection in production before this check existed.

For an athlete under 18, `get_public_signals` and `get_web_threads` leave out everything in the eight brand categories that are for athletes 18 and older, and anything not yet checked. (2026-10-04). A parent or guardian's own connection reads the guardian's own account, which holds none of the athlete's notes, contracts or threads, so it cannot reach the athlete's Web or private data. It is served as under 18 in both tools, so the eight categories are withheld there too (2026-10-05).

There are no delete or send tools at launch. Destructive actions stay in the app behind Face ID.

**Prompt-injection hygiene:**
- Web threads come from public posts, so they are untrusted. They go out as plain data fields, with markup and instruction-like text stripped.
- Every input is validated on the server.
- Logs never contain private content.

## 5. Private data and the unlock

- **The default is public only.** Private access is off for each client until the athlete turns it on in the app with Face ID (device-signed `PUT /mcp/clients/{id}/scopes`).
- **Unlock:** the app posts a grant (`POST /mcp/clients/{id}/grant`, `ttlSeconds` 900 or 3,600; 1 hour is the maximum). The grant lives in API memory only and is never persisted. A restart or scale-out simply means the next private read asks again.
- **Locked read** (built; not yet walked end to end on a phone):
  1. The tool returns "Private data is locked. Open Leave and tap Unlock for ChatGPT."
  2. The server sends `mcp_unlock_request { type, clientId, clientName, provider }` as an alert with an **Unlock** action.
  3. One tap and Face ID in the app post a new grant.
  4. The athlete asks the AI again.
- **Writes always work.** The server wraps the new key, and the share-wrap completes on the next app session.
- Push `mcp_connected { type, clientId, clientName, provider }` refreshes Connections, and it doubles as a security alert: "Leave was connected to ChatGPT. Not you? Disconnect."

- **Disconnect** in the app refuses the client from the next request and deletes its authorization at WorkOS. Only a new sign-in brings it back, with private sharing off, and the athlete is told when it does. **Lock now** ends an unlock on every server instance.

**Athletes aged 13 to 17 (decided 2026-09-23):** they can't share private data with any AI app. The toggle is hidden and the server refuses a private grant for their accounts with `minor_private_blocked`. Claude and Gemini are 18+ anyway. A minor's contracts sit under the parent or guardian relationship, and sending a minor's private data into a third-party chat under that chat's retention rules is not a trade-off Leave will make for them. Leave's minimum age is 13; there is no under-13 path.

## 6. Retention notices (shown word for word on the private toggle; reconfirm against each provider's policy before launch)

- **ChatGPT:** "ChatGPT keeps your chats until you delete them, and OpenAI may use them to train its models unless you turn that off in ChatGPT settings."
- **Claude:** "Claude keeps your chats under Anthropic's consumer terms. If you allow model training in Claude's privacy settings, your chats may be used for training and kept longer."
- **Gemini:** "Gemini stores chats in your Google activity (on by default for 18 months), and human reviewers may read them."
- **Other:** "This AI keeps what you share under its own privacy policy, which Leave can't control."

These notices are about the AI app's own retention. Leave's own model calls (for `add_contract`) run on Gemini 3.8 Flash on Vertex AI, where Google may retain prompts up to 30 days solely for abuse monitoring, as described in `architecture-and-security.md`.

## 7. Build order, and where each step stands

1. **WorkOS config** (section 3). Done; three checks are open.
2. **MCP handler in `api`**: the protected-resource metadata, the token check, and the public read tools. Done and running. Proven from a Claude custom connector.
3. **Writes** (`add_note`, `add_contract`). Built and tested in the server's own tests. Not yet run from an AI app with a real account.
4. **App**: the Connections screen with the private switch, the unlock, Lock now, Disconnect and the two pushes. Built; in testing. `auth.` and `mcp.` stay out of associated domains.
5. **Site**: `leaveyouragent.com/connect`. Published.
6. **Test each client end to end.** Claude: sign-in and public reads done; the private unlock, Lock now and Disconnect not yet walked. ChatGPT, Gemini, Le Chat, Perplexity, Grok, Cursor and VS Code: not tested.
7. **Listings.** Nothing is submitted. A ChatGPT listing comes after ChatGPT has been tested end to end; it also needs OpenAI's verification of the company and domain, a reviewer sign-in that does not depend on an emailed code, and test cases. A Claude Connectors Directory listing waits for funding. Gemini has no third-party directory.
8. **"What your AI can see"**, in plain language. Published on the connect page.

## Decisions (2026-09-23)

1. Connecting happens on the web and grants public data only; private access is a per-client switch in the app, off by default.
2. No custom OAuth scope: clients request `openid offline_access`; access tokens last 5 minutes.
3. Athletes aged 13 to 17 can't share private data with an AI app.
4. An unlock lasts at most 1 hour: 15 minutes by default, or 1 hour, held in memory only.
5. The Claude directory listing waits for funding, because it needs a Claude Team org. Until then, Claude users add Leave as a custom connector through the button or the copy card. This is one click on paid plans. On the Free plan it takes the one custom-connector slot.
6. Leave names ChatGPT, Claude, Gemini and other MCP-capable apps; it does not market clients athletes cannot connect on their own.
7. (2026-10-04) The eight brand categories that are for athletes 18 and older are withheld from an athlete under 18 in AI connections too.

## Sources

MCP authorization 2026-07-28 and security best practices (modelcontextprotocol.io); Claude connector docs, authentication and submission (claude.com/docs/connectors); OpenAI Apps SDK auth and submission (developers.openai.com/apps-sdk); Gemini custom apps (support.google.com/gemini/answer/17209137); Perplexity connectors help; Mistral Le Chat connectors; xAI Grok connectors; Cursor install links; WorkOS AuthKit MCP, custom domains and Magic Auth (workos.com/docs); MCP TypeScript SDK v2 migration; Cloud Run timeouts.
