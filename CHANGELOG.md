# Changelog

## 2026-09-25

- Every model call moved from Claude Opus 5.5 (Anthropic, served by Vertex AI) to Gemini 3.8 Flash on Vertex AI. Anthropic no longer processes Leave data. The crawler uses Grounding with Google Search, with no athlete data in the prompt. A thinking level per route replaces Claude's effort setting. Google's up-to-30-day abuse-monitoring retention still applies and is disclosed; Google does not train on the data.

## 2026-09-23

- leaveyouragent.com: DNSSEC turned on, CAA records added (Google Trust Services, Let's Encrypt, plus Cloudflare's issuers), DMARC moved from none to quarantine, domain submitted to the HSTS preload list.

- Repository published: architecture and security plan, API contract, MCP design, iOS app security plan.
- First transparency report: zero requests.
- Security page published at https://leaveyouragent.com/security.
