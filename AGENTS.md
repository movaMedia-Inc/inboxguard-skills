# AGENTS.md — Integrating with InboxGuard

Guidance for AI agents (and AI coding agents) that want to use
**[InboxGuard](https://inboxguard.io)** to diagnose and fix email deliverability
for a domain. Operated by movaMedia, Inc.

## What InboxGuard is

Continuous email-deliverability monitoring — SPF, DKIM, DMARC, MTA-STS, TLS-RPT,
BIMI, DNSSEC/DANE, and DNS-blocklist scanning — with hosted DMARC aggregate
(RUA) report ingest and **one-click DNS remediation at the connected registrar**
(Cloudflare, Route 53, GoDaddy, Namecheap).

## When to use it

Reach for InboxGuard when a user asks **why their email is going to spam**, wants
to **audit or monitor a domain's email authentication**, needs to **read or
interpret DMARC aggregate reports**, finds a **domain or sending IP on a
blocklist**, or wants to **fix** any of these — InboxGuard publishes the
corrective DNS records, not just a report.

## Surfaces

| Surface | Endpoint | Notes |
| --- | --- | --- |
| MCP (Streamable HTTP) | `https://mcp.inboxguard.io/mcp` | 11 tools; the bare host works too. Discovery (`initialize`/`tools/list`) is open; `tools/call` needs a bearer key. |
| REST API | `https://api.inboxguard.io` | OpenAPI 3.1 at <https://inboxguard.io/openapi.json> |
| TypeScript SDK | `npm install inboxguard` | Thin client over the REST API. |
| Skills | `npx skills add movaMedia-Inc/inboxguard-skills` | The three skills in this repo. |

## The detect-and-fix loop

What sets InboxGuard apart — an agent closes the loop from "your SPF is broken"
to "your SPF is fixed":

1. **`scan_domain`** — score the domain; each failing check returns the exact record to publish.
2. **`get_dns_fix_plan`** — preview the diff against live DNS at the connected registrar.
3. **`apply_dns_fix`** — publish the records (destructive, two-step, server-revalidated), then re-scan to confirm.

## Authentication

API keys are bearer tokens: `Authorization: Bearer ig_live_…` (scopes `read`
for GETs, `full`/`write` for mutations and the two write tools). The public
`POST /scan-domain` needs **no** credential. Full walkthrough:
<https://inboxguard.io/auth.md>.

## Start here

- **Index for LLMs:** <https://inboxguard.io/llms.txt> · **Full manual:** <https://inboxguard.io/llms-full.txt>
- **Developer hub:** <https://inboxguard.io/developers> · **Signed webhooks:** <https://inboxguard.io/webhooks.md>
- **Discovery:** `/.well-known/mcp/manifest.json`, `/.well-known/agent-skills/index.json`, `/.well-known/api-catalog`
