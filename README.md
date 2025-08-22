# Rello — Full Docs Bundle (v2)
Date: 2025-08-22 · Timezone: Europe/Warsaw

This repository folder (zip) aggregates **all current documents** from the Master Checklist so any chat/team can read the same source of truth.

## Structure
- `/product` — product scope, commands, dialog flows, tiers, arbitration.
- `/tech` — DB schema, API contracts, webhooks/events, deeplink/JWT, payouts engine, deposits/limits, admin console.
- `/ops` — runbooks, monitoring/alerts, wallet SOP, backup/restore, incident response.
- `/quality_launch` — test plan, launch plan.
- `/bots` — i18n texts RU/EN.
- `/legal` — Terms/Privacy/MOU drafts.
- `master_checklist_v2.md` — status overview.

**Start here:** `product/features_commands_v2.md` → `tech/db_schema_v2.md` → `tech/api_contracts_v1.md` → `ops/sop_wallets_v1.md` → `quality_launch/test_plan_v1.md`.

## How to use
1. Unzip to a private repo (e.g., `rello-docs`) and push.
2. Share read-only link between chats and pin it.
3. Any edits: bump minor version in the file name (e.g., `_v2` → `_v2.1`).
