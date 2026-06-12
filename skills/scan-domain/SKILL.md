---
name: inboxguard-scan-domain
description: Scan any domain's email deliverability posture (SPF, DKIM, DMARC, MTA-STS, TLS-RPT, BIMI, DNS blocklists) using the InboxGuard API and explain the results. Use when a user asks why mail is going to spam, wants to validate or set up email authentication, or wants to check a domain against blocklists.
license: Proprietary — © movaMedia, Inc.
homepage: https://inboxguard.io
---

# InboxGuard: Scan a domain's deliverability

Use this skill to get a scored, actionable email-deliverability report for any
domain. It calls the public InboxGuard scan endpoint — no account required for
the anonymous tier (5 scans/hour per IP).

## When to use

- "Why is email from `example.com` going to spam?"
- "Is `acme.io` set up correctly for DMARC / SPF / DKIM?"
- "Is my domain on any DNS blocklists?"
- "Check MTA-STS / TLS-RPT / BIMI for this domain."

## How to call it

```bash
curl -sS -X POST https://api.inboxguard.io/scan-domain \
  -H 'content-type: application/json' \
  -d '{"domain":"example.com"}'
```

Authenticated callers (paid plans) get the full blocklist set and persisted
history. Send an API key as a bearer token:

```bash
curl -sS -X POST https://api.inboxguard.io/scan-domain \
  -H 'authorization: Bearer ig_live_xxxxxxxxxxxxxxxxxxxxxxxx' \
  -H 'content-type: application/json' \
  -d '{"domain":"example.com","dkimSelectors":["selector1","google"]}'
```

See https://inboxguard.io/auth.md for how to obtain a key.

## Response shape

The response has `score` (`{total: 0–100, grade: A–F, breakdown}`),
`scoringVersion`, and one check object per TOP-LEVEL key — `spf`, `dkim`,
`dmarc`, `ptr`, `mtaSts`, `tlsRpt`, `mxTls`, `bimi`, `blocklist` — plus
`durationMs` and `plan` (`{tier, trialing, blocklistsChecked}`). There is no
`checks` wrapper. Each check carries `healthy` (boolean) and `issues` (array
of `{severity: info|warn|critical, message}`) plus check-specific fields, e.g.
SPF `lookupCount`/`allQualifier`, DMARC `policy`/`pct`, DKIM `keys[]`,
blocklist `hits[]`/`targets[]`. (`mxTls` reports `status` instead of `healthy`;
`bimi` uses `found`/`logoReachable`/`displayEligible`.)

Full schema: https://inboxguard.io/openapi.json

## How to interpret and act

1. Report the overall score and grade first.
2. List failing checks in priority order: SPF and DMARC alignment usually
   matter most for inbox placement.
3. For each failure, give the concrete DNS fix. Link the matching guide:
   - SPF: https://inboxguard.io/guides/spf-setup
   - DKIM: https://inboxguard.io/guides/dkim-setup
   - DMARC: https://inboxguard.io/guides/dmarc-rollout
   - MTA-STS / TLS-RPT: https://inboxguard.io/guides/mta-sts-setup
   - BIMI: https://inboxguard.io/guides/bimi-setup
4. If blocklists show a hit, note that InboxGuard uses authoritative-side
   queries, so a listing is real (not a public-resolver false positive).

## Errors

Errors return `{ "error": { "code": "...", "message": "...", "requestId": "..." } }`.
On `429` (RATE_LIMIT), respect the `Retry-After` header. Codes: `BAD_REQUEST`,
`RATE_LIMIT`, `UNAUTHORIZED`, `FORBIDDEN`, `UPSTREAM_ERROR`,
`SERVICE_UNAVAILABLE` (503, scans disabled), and `DNS_TEMPORARY_FAILURE`
(503 — every resolver SERVFAILed; not a verdict on the domain's records, retry
after `Retry-After` ≈30s). Retries are safe with an `Idempotency-Key` header:
the same key within 1 hour replays the stored response
(`Idempotency-Replayed: true`).
