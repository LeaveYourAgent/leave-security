# Changelog

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
