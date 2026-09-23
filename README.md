# Leave: security and architecture, in public

Leave helps athletes be their own agent. It asks athletes to trust it with their contracts and the things they'd tell no one else, so we publish how it works: the architecture, the encryption design, the API contract, and the AI chat (MCP) design. Nothing here is secret, and none of the controls depend on the design being hidden.

The plain-language version, with what is live today, is at **https://leaveyouragent.com/security**.

## Status

Status as of **September 23, 2026**: Leave has not launched. The production backend is designed but not built, the iOS app is in testing on TestFlight with fictional data only, and the only personal data Leave holds is waitlist email addresses. Each document marks what is built, in testing, or planned. When that changes, the documents change here, with history.

## What's here

| File | What it covers |
|---|---|
| [docs/architecture-and-security.md](docs/architecture-and-security.md) | The Google Cloud architecture, organization policies, network and abuse controls, the two-part encryption design, and the security review with every hole found and how it was closed |
| [docs/api-contract.md](docs/api-contract.md) | The API between the app and the backend, including key exchange, session grants, and device approval |
| [docs/mcp.md](docs/mcp.md) | How Leave connects to AI chats (ChatGPT, Claude, Gemini) through the Model Context Protocol, public by default |
| [docs/ios-app-security.md](docs/ios-app-security.md) | The iOS app's side: the crypto module, the screens, and what is built today |
| [transparency/](transparency/) | Transparency reports: requests for user data and how we answered |
| [SECURITY.md](SECURITY.md) | How to report a vulnerability, and our safe-harbor promise |
| [CHANGELOG.md](CHANGELOG.md) | What changed, and when |

What's deliberately not here: business financials (costs, margins, budgets) and anything that would only help an attacker, like account IDs and signing identities.

## Questions

chris@leaveyouragent.com
