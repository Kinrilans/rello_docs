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
@@
+# Инструкция: как пользоваться репозиторием `rello_docs` в новом dev-чате (v1)
+
+Репозиторий: https://github.com/Kinrilans/rello_docs.git  
+Часовой пояс: Europe/Warsaw · Активы: USDT/USDC · Сеть старта: TRC-20
+
+> Источник истины — файлы в ветке `main`. Если в чате возникает расхождение, ссылаться на конкретный файл/раздел и предлагать правку (патч в Markdown).
+> Формат ответов в чате по коду: готовые файлы/дифф + шаги запуска/проверки + ссылки на разделы.  
+> Минорные правки документа — `_v2` → `_v2.1`; мажор — новый файл `_v3`, старый помечается как `archive`.
+
+---
+
+## 0) Правила работы с документами
+- Источник истины — файлы в `main`.
+- В ответах — точные ссылки на файлы/разделы.
+- На несоответствия — присылать **патч** в Markdown с изменениями.
+- Ответы ассистента: **код/дифф + команды запуска + ссылки на документы**.
+
+---
+
+## 1) Корень репозитория
+- **`README.md`** — оглавление и «порядок чтения».  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/README.md
+- **`master_checklist_v2.md`** — перечень артефактов и статусы.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/master_checklist_v2.md
+- **`.gitignore` / `CONTRIBUTING.md`** — политика коммитов/PR, запрет секретов.  
+  RAW (.gitignore): https://raw.githubusercontent.com/Kinrilans/rello_docs/main/.gitignore  
+  RAW (CONTRIBUTING): https://raw.githubusercontent.com/Kinrilans/rello_docs/main/CONTRIBUTING.md
+
+---
+
+## 2) `product/` — продукт/UX
+- **`product/vision_positioning_v1.md`** — визия/позиционирование.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/vision_positioning_v1.md
+- **`product/features_commands_v2.md`** — каталог функций/команд по ботам/ролям.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/features_commands_v2.md
+- **`product/flows_common_v2.md`** — UX-принципы (шаги, CTA, back, deeplink).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/flows_common_v2.md
+- **`product/flows_dealdesk_v2.md`** — офферы/сделки, pass-through/EOD (для @DealDeskBot).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/flows_dealdesk_v2.md
+- **`product/flows_clubgate_v2.md`** — онбординг компании, доверительные пары (для @ClubGateBot).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/flows_clubgate_v2.md
+- **`product/flows_treasury_v2.md`** — депозиты, выплаты, EOD-неттинг (для @TreasuryBot).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/flows_treasury_v2.md
+- **`product/flows_arbiter_v2.md`** — споры и решения арбитра (для @ArbiterDeskBot).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/flows_arbiter_v2.md
+- **`product/tiers_table_v2.md`** — tier S/M/L/XL, k-фактор, cap’ы (per-deal/day/exposure).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/tiers_table_v2.md
+- **`product/arbitration_rules_v2.md`** — правила удержаний/сроков (1–2 страницы).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/arbitration_rules_v2.md
+- **`product/user_flows_v1.md`** — user flow всех существующих функций  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/user_flows_v1.md
+
+---
+
+## 3) `tech/` — технические спецификации
+- **`tech/db_schema_v2.md`** — сущности/таблицы, enum’ы, индексы.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/db_schema_v2.md
+- **`tech/api_contracts_v1.md`** — REST: Offers/Deals/Treasury/Trust/Arbiter/Webhooks.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/api_contracts_v1.md  
+  _Важно_: денежные операции → `Idempotency-Key` + (иногда) `X-2FA-Code`.
+- **`tech/webhooks_events_v1.md`** — модель доставки, заголовки `X-*`, HMAC, ретраи, payload.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/webhooks_events_v1.md
+- **`tech/deeplink_jwt_v1.md`** — Short-Token/JWT для безопасных deeplink’ов.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/deeplink_jwt_v1.md
+- **`tech/engine_payouts_v1.md`** — state-machine выплат, cap per-tx, батчинг.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/engine_payouts_v1.md
+- **`tech/deposits_limits_v1.md`** — лимиты, экспозиция, ledger-операции, T+N.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/deposits_limits_v1.md
+- **`tech/admin_console_v1.md`** — конфигурация сетей/кошельков, лимиты, safe-mode, webhooks.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/admin_console_v1.md
+
+---
+
+## 4) `ops/` — операции и безопасность
+- **`ops/runbooks_v1.md`** — ежедневки, инциденты, ребаланс.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/ops/runbooks_v1.md
+- **`ops/monitoring_alerts_v1.md`** — метрики/SLI/SLO, правила алёртов.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/ops/monitoring_alerts_v1.md
+- **`ops/sop_wallets_v1.md`** — hot/deposit/cold, подпись, allow-list, лимиты, safe-mode.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/ops/sop_wallets_v1.md
+- **`ops/backup_restore_v1.md`** — RPO/RTO, PITR, расписания, тест-восстановления.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/ops/backup_restore_v1.md
+- **`ops/incident_response_plan_v1.md`** — SEV1–3, роли, плейбуки, коммуникации, постмортем.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/ops/incident_response_plan_v1.md
+
+---
+
+## 5) `quality_launch/` — тесты и запуск
+- **`quality_launch/test_plan_v1.md`** — E2E P0 (pass-through, EOD), негатив/границы, Go/No-Go.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/quality_launch/test_plan_v1.md
+- **`quality_launch/launch_plan_v1.md`** — Pre-flight → Pilot → Canary → Full, T-7…T+7.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/quality_launch/launch_plan_v1.md
+
+---
+
+## 6) `bots/` — тексты и локализация
+- **`bots/bot_texts_ru_en_v1.md`** — словарь i18n (RU/EN), плейсхолдеры, кнопки, ошибки.  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/bots/bot_texts_ru_en_v1.md
+
+---
+
+## 7) `legal/` — черновики
+- **`legal/terms_draft_v1.md`** — условия клуба (draft).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/legal/terms_draft_v1.md
+- **`legal/privacy_draft_v1.md`** — приватность (draft).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/legal/privacy_draft_v1.md
+- **`legal/mou_draft_v1.md`** — MOU (draft).  
+  RAW: https://raw.githubusercontent.com/Kinrilans/rello_docs/main/legal/mou_draft_v1.md
+
+---
+
+## 8) Как формулировать задачи в новом чате
+Шаблон:
+```
+[Задача] Коротко
+Контекст: ссылки на разделы из rello_docs (1–2)
+Что сделать: кратко (ожидание результата)
+Материалы: файл/дифф (если правим код/SQL)
+Проверка: команды и шаги
+```
+Требование к ответам ассистента: **код/дифф + команды запуска + ссылки на конкретные разделы**.
+
+---
+
+## 9) Частые маршруты чтения
+- Watcher TRC-20 → `tech/webhooks_events_v1.md` (pay_in.detected), `tech/db_schema_v2.md` (incoming_tx), `tech/api_contracts_v1.md` (treasury), `ops/monitoring_alerts_v1.md` (метрики).  
+  RAW:  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/webhooks_events_v1.md  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/db_schema_v2.md  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/api_contracts_v1.md  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/ops/monitoring_alerts_v1.md
+- EOD-выплата → `product/flows_treasury_v2.md`, `tech/engine_payouts_v1.md`, `tech/api_contracts_v1.md` (trust/settlements), `ops/runbooks_v1.md`.  
+  RAW:  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/flows_treasury_v2.md  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/engine_payouts_v1.md  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/api_contracts_v1.md  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/ops/runbooks_v1.md
+- Лимиты/депозиты → `product/tiers_table_v2.md`, `tech/deposits_limits_v1.md`, `tech/api_contracts_v1.md` (limits).  
+  RAW:  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/product/tiers_table_v2.md  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/deposits_limits_v1.md  
+  • https://raw.githubusercontent.com/Kinrilans/rello_docs/main/tech/api_contracts_v1.md
+
+---
+
+## 10) Если документа не хватает
+- Сослаться на ближайший раздел и предложить правку.
+- Прислать патч в стиле diff/Markdown, имя по правилам версий (`*_v2.1.md` или новый `*_v3.md`).
+- При желании обновить `master_checklist_v2.md` статусом (необязательно).
