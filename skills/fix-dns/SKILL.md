---
name: inboxguard-fix-dns
description: Detect AND fix a domain's email-DNS problems end to end with InboxGuard — scan, preview the exact record changes, apply them at the connected registrar, then re-scan to confirm. Use when a user wants you to not just report SPF/DKIM/DMARC/MTA-STS/TLS-RPT issues but actually remediate them, or asks "fix my email DNS".
license: Proprietary — © movaMedia, Inc.
homepage: https://inboxguard.io
---

# InboxGuard: Fix a domain's email DNS (scan → plan → apply → verify)

Use this skill to *remediate* email-DNS problems, not just diagnose them. An
agent can take a domain from "failing" to "fixed" without leaving the loop:
scan it, preview the exact DNS-record changes, apply them at the customer's
connected registrar, then re-scan to confirm.

This is the find-and-fix surface that sits on top of [scanning](https://inboxguard.io/skills/scan-domain/SKILL.md).
It is authenticated and account-scoped: the domain must be tracked in the
InboxGuard org and a registrar (Cloudflare / Route 53 / GoDaddy / Namecheap)
must be connected for the zone.

## When to use

- "Fix the SPF/DMARC/MTA-STS problems on `example.com`."
- "My scan found issues — go ahead and publish the records."
- "Set up a DMARC monitoring policy for `acme.io`."
- After a `scan_domain` that returned `remediations` with `autoFixable: true`.

If the user only wants a diagnosis (no DNS writes), use the
[scan-domain skill](https://inboxguard.io/skills/scan-domain/SKILL.md) instead.

## The safety model (read this first)

`apply_dns_fix` (`POST /domains/{id}/dns-apply`) **writes DNS records at the
customer's registrar** — it is destructive. Four guardrails make it safe:

1. **Two-step.** You cannot apply blind. First call `get_dns_fix_plan` to get a
   `connectionId` and an `ops` array, then pass *both* to `apply_dns_fix`
   verbatim.
2. **Server-revalidated.** The server re-derives the diff from the latest scan
   and rejects any op that no longer matches (`400`, stale plan). An agent can
   **never apply arbitrary records** — only the changes InboxGuard itself would
   compute. If DNS changed since the preview, re-run `get_dns_fix_plan`.
3. **Owner/admin write key.** Applying needs an API key with `write`/`full`
   scope **and** an owner or admin org role (`403` otherwise). Previewing only
   needs read.
4. **Not everything auto-fixes.** Only changes InboxGuard can publish without a
   human decision are in `ops` — a *missing* DMARC record, the MTA-STS version
   TXT, and TLS-RPT. SPF sender lists, DKIM keys, and BIMI logos need customer
   input and come back as `manualReview` (and as `autoFixable: false`
   remediations on the scan). Show those to the user; don't try to publish them.

## The loop

### 1. Scan — and read `remediations`

Every scan now returns two extra top-level fields:

- `remediations` — an ordered (critical-first) list of concrete fixes. Each has
  `check`, `code`, `severity`, `title`, `summary`, `steps[]`, an optional
  `record` (the exact DNS record to publish), and `autoFixable` (whether the
  registrar engine can apply it).
- `dnssec` — `{ dsPresent, dnskeyPresent, enabled, dane[], daneEnabled, ... }`.
  `enabled` is true only when the zone is signed **and** anchored (DS + DNSKEY);
  `dane[]` reports per-MX DANE/TLSA detection.

MCP:

```jsonc
// tools/call → scan_domain
{ "name": "scan_domain", "arguments": { "domain": "example.com" } }
// structuredContent includes: score, grade, checks, remediations[], dnssec
```

If every `autoFixable` remediation is already clean, there's nothing to apply —
report the `manualReview`-style items and stop.

### 2. Preview the plan — `get_dns_fix_plan`

```jsonc
// tools/call → get_dns_fix_plan
{ "name": "get_dns_fix_plan", "arguments": { "domain": "example.com" } }
```

Returns (the fields you need to carry forward in **bold**):

```jsonc
{
  "domain": "example.com",
  "connected": true,
  "connectionId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", // ← pass to apply
  "provider": "cloudflare",
  "apex": "example.com",
  "ops": [                                                  // ← pass to apply
    {
      "kind": "create",
      "reason": "Publish a DMARC monitoring policy",
      "before": null,
      "after": { "name": "_dmarc.example.com", "type": "TXT",
                 "value": "v=DMARC1; p=none; rua=mailto:rua@example.com; fo=1", "ttl": 3600 }
    }
  ],
  "opCount": 1,
  "manualReview": [ /* SPF/DKIM/BIMI items needing a human decision */ ],
  "reason": null,
  "nextStep": "Call apply_dns_fix with this connectionId and ops ..."
}
```

Handle the not-ready cases before applying:

- `connected: false` → no registrar is connected. Tell the user to connect one
  in **Settings → Integrations**; stop.
- `connected: true` but `ops` empty + a `reason` → no connected registrar
  manages this zone, or there's nothing auto-fixable. Surface `manualReview` and
  stop.
- No scan yet (`400`) → run `scan_domain` first, then retry.

Show the user what will change (the `ops`) before you apply.

### 3. Apply — `apply_dns_fix` (destructive)

Pass the `connectionId` and the `ops` array back **unchanged**:

```jsonc
// tools/call → apply_dns_fix
{
  "name": "apply_dns_fix",
  "arguments": {
    "domain": "example.com",
    "connectionId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "ops": [ /* the exact ops array from get_dns_fix_plan */ ]
  }
}
```

The result reports each op's outcome (it is **not** transactional — partial
success is possible):

```jsonc
{
  "planId": "…",
  "connectionId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "provider": "cloudflare",
  "apex": "example.com",
  "results": [
    { "kind": "create", "name": "_dmarc.example.com", "type": "TXT", "ok": true, "stepId": "…" }
  ]
}
```

Inspect `results`: report any `ok: false` (with its `error`) and retry just
those, or fall back to the manual steps.

### 4. Verify — re-scan

Run `scan_domain` again. Confirm the score went up and the fixed checks now pass.
DNS can take time to propagate (PTR/registrar TTLs especially) — if a record was
just written but the re-scan still shows it missing, wait and re-scan.

## Calling over REST (instead of MCP)

The MCP tools wrap two endpoints. Resolve the domain NAME to its id via
`GET /domains` (match `domain` case-insensitively), then:

```bash
# Preview (read scope is enough)
curl -sS https://api.inboxguard.io/domains/$DOMAIN_ID/dns-diff \
  -H 'authorization: Bearer ig_live_xxxxxxxxxxxxxxxxxxxxxxxx'

# Apply (owner/admin + full scope) — body is connectionId + the ops verbatim
curl -sS -X POST https://api.inboxguard.io/domains/$DOMAIN_ID/dns-apply \
  -H 'authorization: Bearer ig_live_xxxxxxxxxxxxxxxxxxxxxxxx' \
  -H 'content-type: application/json' \
  -d '{"connectionId":"7c9e6679-7425-40de-944b-e07fc1f90ae7","ops":[ /* from dns-diff */ ]}'
```

Full schema: https://inboxguard.io/openapi.json (see `getDnsDiff` / `applyDnsFix`).
See https://inboxguard.io/auth.md for how to obtain a key.

## Errors

Errors return `{ "error": { "code": "...", "message": "...", "requestId": "..." } }`.
The ones specific to this loop:

- `400` on `dns-diff` → no scan exists yet; run `scan_domain` first.
- `400` on `dns-apply` → the plan is **stale** (DNS changed since the preview).
  Re-run `get_dns_fix_plan` and apply the fresh `ops`.
- `403` on `dns-apply` → the key lacks `write`/`full` scope or the caller isn't
  an owner/admin. Previewing still works with a read key.
- `404` → the domain isn't tracked, or the registrar connection wasn't found.

Honor `Retry-After` on `429`/`503` as with any InboxGuard endpoint.
