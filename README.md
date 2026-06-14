# InboxGuard — agent skills & integration guide

Public [agent skills](https://skills.sh) and integration docs for
**[InboxGuard](https://inboxguard.io)** — continuous email-deliverability
monitoring (SPF, DKIM, DMARC, MTA-STS, TLS-RPT, BIMI, DNS blocklists) with
hosted DMARC aggregate-report ingest and one-click DNS remediation.

## Install the skills

```bash
npx skills add movaMedia-Inc/inboxguard-skills
```

Installs three skills:

| Skill | What it does |
| --- | --- |
| `inboxguard-scan-domain` | Scan any domain's deliverability posture (SPF/DKIM/DMARC/MTA-STS/TLS-RPT/BIMI/blocklists) and explain the results. |
| `inboxguard-dmarc-monitoring` | Stage a safe DMARC rollout (none → quarantine → reject) and interpret aggregate (RUA) reports. |
| `inboxguard-fix-dns` | Detect **and fix** email-DNS problems end to end: scan → preview the diff → apply at the registrar → re-scan. |

## Use InboxGuard directly

- **MCP server (Streamable HTTP):** `https://mcp.inboxguard.io/mcp` — 18 tools; discovery is open, `tools/call` needs a bearer API key.
- **REST API:** `https://api.inboxguard.io` · **OpenAPI 3.1:** <https://inboxguard.io/openapi.json>
- **TypeScript SDK:** `npm install inboxguard`
- **Auth:** <https://inboxguard.io/auth.md> · **Developer hub:** <https://inboxguard.io/developers>

One-shot scan, no account (rate-limited 5/hour per IP):

```bash
curl -sS -X POST https://api.inboxguard.io/scan-domain \
  -H 'content-type: application/json' -d '{"domain":"example.com"}'
```

See **[AGENTS.md](./AGENTS.md)** for full integration guidance and the
detect-and-fix loop.

---
© movaMedia, Inc. · <https://inboxguard.io>
