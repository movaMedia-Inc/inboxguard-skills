---
name: inboxguard-dmarc-monitoring
description: Set up and interpret DMARC for a sending domain using InboxGuard — staged rollout (none → quarantine → reject), hosted aggregate-report ingest, and per-source-IP analysis. Use when a user wants to deploy DMARC safely, read DMARC aggregate (RUA) reports, or find which senders are failing authentication.
license: Proprietary — © movaMedia, Inc.
homepage: https://inboxguard.io
---

# InboxGuard: DMARC rollout and monitoring

Use this skill to help a user deploy DMARC without blocking legitimate mail, and
to interpret the aggregate reports InboxGuard ingests on their behalf.

## When to use

- "How do I set up DMARC for `example.com`?"
- "What does this DMARC aggregate report mean?"
- "Which IPs are failing DMARC for my domain?"
- "Is it safe to move my DMARC policy to `p=reject`?"

## Staged rollout

1. **Publish `p=none`** to start collecting reports without affecting delivery:
   `v=DMARC1; p=none; rua=mailto:<your InboxGuard RUA mailbox>`
2. **Watch aggregate reports** until all legitimate senders show SPF or DKIM
   alignment pass.
3. **Move to `p=quarantine`** (optionally `pct=25` and ramp up).
4. **Move to `p=reject`** once aligned pass rates are stable.

Full guide: https://inboxguard.io/guides/dmarc-rollout
Reading reports: https://inboxguard.io/guides/reading-dmarc-reports

## Reading reports via the API

The endpoint is nested under the domain: first resolve the domain's id from
`GET /domains` (match the `domain` field), then fetch its reports
(`days` 1–90, default 30):

```bash
DOMAIN_ID=$(curl -sS https://api.inboxguard.io/domains \
  -H 'authorization: Bearer ig_live_xxxxxxxxxxxxxxxxxxxxxxxx' \
  | jq -r '.items[] | select(.domain=="example.com") | .id')

curl -sS "https://api.inboxguard.io/domains/$DOMAIN_ID/dmarc-reports?days=30" \
  -H 'authorization: Bearer ig_live_xxxxxxxxxxxxxxxxxxxxxxxx'
```

The response has `ruaInbox` (the hosted `rua-…@reports.inboxguard.io` address
plus a ready-to-publish `dmarcSnippet`), `daily[]` per-day/per-reporter rollups
(`date`, `source`, `spfAligned`, `dkimAligned`, `passCount`, `failCount`),
`topSenders[]` (`sourceIp`, `headerFrom`, `count`, `passCount`, `failCount`),
and window `totals` (`volume`, `pass`, `fail`, `sources`). Read `topSenders` to
find unauthenticated senders — a high `count` with a high `failCount` is either
a forgotten legitimate sender (fix its auth) or spoofing (good that DMARC will
block it). Requires a plan with DMARC ingest (403 `FORBIDDEN` otherwise). Over
MCP, the `get_dmarc_summary` tool does the id resolution for you by domain name.

Authentication setup: https://inboxguard.io/auth.md ·
Schema: https://inboxguard.io/openapi.json
