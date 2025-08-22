# Схема БД и сущностей (v2)

USDT/USDC только · Сети: ERC‑20/TRC‑20 · Мульти‑бот (ClubGate/DealDesk/Treasury/Arbiter) · Добавлены **доверительные пары** и **EOD‑неттинг**.

**Тех. база:** PostgreSQL 15+, UUID v4 как PK, временные метки `created_at/updated_at TIMESTAMPTZ`. Денежные поля: `numeric(38, 8)` в нативной единице токена + зеркальные `*_usd numeric(38, 2)` для метрик. Все суммы стаблов ~1:1 к USD.

---

## 0) Конвенции и общие таблицы
- PK: `id uuid default gen_random_uuid()`.
- FK: `ON UPDATE CASCADE ON DELETE RESTRICT` (кроме файлов/логов — `SET NULL`).
- Аудит: критичные действия → `audit_log` (append‑only), финансы → отдельные **ledger**‑журналы (append‑only).

**network**
- `id`, `code` (ETH|TRON), `name`, `min_confirmations int`, `tx_cap_usd numeric`, `is_enabled bool`.

**token**
- `id`, `symbol` (USDT|USDC), `name`, `decimals int`, `is_enabled bool`.

**network_token**
- `id`, `network_id fk`, `token_id fk`, `contract text`, `decimals int`, `is_enabled bool`.
- Unique: `(network_id, token_id)`.

**tier**
- `id`, `code` (S|M|L|XL), `min_deposit_usd`, `k_base numeric(8,3)`, `deal_cap_pct int`, `daily_payout_cap_pct int`, `open_exposure_cap_pct int`, `withdraw_window_days int`, `dispute_hold_pct int`.

**fee_policy**
- `id`, `maker_bps int`, `taker_bps int`, `arbitration_fee_bps int`, `effective_from timestamptz`.

---

## 1) Компании и пользователи
**company**
- `id`, `name`, `slug`, `status` (active|suspended|blocked), `tier_id fk`, `k_base numeric(8,3)`, `risk_score int`, `work_hours jsonb`, `timezone text`, `created_at`.
- Unique: `(slug)`.

**member**
- `id`, `company_id fk`, `telegram_id bigint`, `display_name`, `status` (active|disabled), `is_2fa_enabled bool`, `created_at`.
- Unique: `(telegram_id)`.

**role_assignment**
- `id`, `member_id fk`, `role` (company_admin|trader|approver|arbiter|admin_platform), `created_at`.
- Unique: `(member_id, role)`.

**invite**
- `id`, `code`, `company_id fk NULL`, `role_default`, `expires_at`, `max_uses int`, `used_count int`.
- Unique: `(code)`.

---

## 2) Реквизиты и allow‑list
**bank_account**
- `id`, `company_id fk`, `currency char(3)`, `account_name`, `iban_or_number`, `bank_name`, `swift_bic`, `routing`, `is_default bool`, `label`, `created_at`.
- Unique: `(company_id, iban_or_number)`.

**crypto_wallet_template**
- `id`, `company_id fk`, `network_id fk`, `token_id fk`, `address`, `memo_tag`, `label`, `is_verified bool`, `created_at`.
- Unique: `(company_id, network_id, token_id, address)`.

**address_allowlist** (для выплат)
- `id`, `company_id fk`, `network_id fk`, `token_id fk`, `address`, `label`, `created_at`.
- Unique: `(company_id, network_id, token_id, address)`.

---

## 3) Лимиты и снапшоты
**company_limit_snapshot**
- `id`, `company_id fk`, `limit_usd`, `deal_cap_usd`, `daily_payout_cap_usd`, `open_exposure_cap_usd`, `as_of`.

**exposure_snapshot**
- `id`, `company_id fk`, `open_exposure_usd`, `daily_payout_used_usd`, `trust_reserve_usd`, `as_of`.

**trader_limit** (внутренние лимиты компании)
- `id`, `company_id fk`, `member_id fk`, `max_single_usd`, `max_daily_usd`, `created_at`.
- Unique: `(company_id, member_id)`.

---

## 4) Офферы и подписки
**offer**
- `id`, `company_id fk`, `direction` (cash_in|cash_out), `mode` (pass_through|deposit), `network_id fk`, `token_id fk`, `fiat_currency char(3)`, `amount_min`, `amount_max`, `price_type` (fixed|corridor|markup), `price_value numeric(18,8)`, `corridor_min`, `corridor_max`, `markup_bps int`, `deadline_hours int`, `status` (draft|active|paused|expired|closed), `created_at`.
- Index: `(status, direction, token_id, network_id)`.

**watch_subscription**
- `id`, `member_id fk`, `direction NULL`, `token_id NULL`, `network_id NULL`, `partner_company_id NULL`, `created_at`.

---

## 5) Сделки и события
**deal**
- `id`, `offer_id fk NULL`, `initiator_company_id fk`, `counterparty_company_id fk`, `direction`, `mode`, `network_id fk`, `token_id fk`, `fiat_currency`, `amount_token`, `amount_usd`, `rate_fiat_per_token numeric(18,8)`, `deadline_at`, `state` (proposed|funded|in_progress|completed|cancelled|disputed|resolved), `is_trusted_netting bool default false`, `trust_session_id fk NULL`, `created_by_member_id fk`, `created_at`, `closed_at NULL`.
- Index: `(state)`, `(initiator_company_id, state)`, `(counterparty_company_id, state)`.

**deal_event** (таймлайн)
- `id`, `deal_id fk`, `type` (fiat_sent|fiat_received|crypto_sent|crypto_received|auto_payout_sent|deadline_extended|cancel_requested|cancelled|dispute_opened|dispute_resolved|trust_flagged|trust_removed), `payload jsonb`, `member_id fk NULL`, `created_at`.
- Index: `(deal_id, created_at)`.

**deal_chat_message**
- `id`, `deal_id fk`, `member_id fk`, `text`, `attachments jsonb`, `created_at`.

---

## 6) Казначейство и кошельки
**platform_wallet**
- `id`, `network_id fk`, `token_id fk`, `address`, `label`, `type` (hot|cold), `daily_payout_cap_usd`, `is_active`.

**incoming_tx**
- `id`, `network_id fk`, `token_id fk`, `tx_hash`, `from_address`, `to_address`, `amount_token`, `amount_usd`, `confirmations int`, `status` (seen|confirmed|failed), `deal_id fk NULL`, `deposit_id fk NULL`, `created_at`.
- Unique: `(network_id, tx_hash)`; Index: `(to_address)`, `(deal_id)`.

**outgoing_tx**
- `id`, `network_id fk`, `token_id fk`, `tx_hash NULL`, `from_wallet_id fk`, `to_address`, `amount_token`, `amount_usd`, `status` (queued|signed|broadcast|confirmed|failed|partial), `deal_id fk NULL`, `payout_request_id fk NULL`, `eod_settlement_id fk NULL`, `idempotency_key text`, `created_at`.
- Unique: `(idempotency_key)`.

**deposit**
- `id`, `company_id fk`, `network_id fk`, `token_id fk`, `address`, `balance_token`, `balance_usd`, `status` (active|withdraw_pending|closed), `created_at`.

**deposit_ledger** (append‑only)
- `id`, `company_id fk`, `deposit_id fk NULL`, `deal_id fk NULL`, `dispute_id fk NULL`, `type` (fund|hold|release|penalty|refund|adjustment|trust_reserve|trust_release), `amount_token` (знак ±), `amount_usd`, `notes`, `created_at`.
- Index: `(company_id, created_at)`, `(deposit_id, created_at)`.

**payout_request**
- `id`, `company_id fk`, `network_id fk`, `token_id fk`, `to_address`, `amount_token`, `amount_usd`, `status` (pending|approved|sent|failed|cancelled), `approved_by_member_id fk NULL`, `created_at`.

---

## 7) Доверительные пары и EOD‑неттинг (новое)
**trust_pair**
- `id`, `company_a_id fk`, `company_b_id fk`, `status` (proposed|active|paused|disabled), `tokens text[]` (из {USDT,USDC}), `networks text[]` (из {ERC20,TRC20}), `daily_credit_limit_usd`, `reserve_pct int`, `cutoff_local_time time`, `timezone text` (IANA), `created_by fk`, `created_at`.
- Нормализация пары: хранить `company_low_id`, `company_high_id` и Check‑constraint `company_low_id < company_high_id`.
- Unique: `(company_low_id, company_high_id)`.

**trust_session**
- `id`, `pair_id fk`, `session_date date` (в TZ пары), `state` (open|closed|settled|disputed), `net_token` (USDT|USDC), `net_amount_token`, `net_usd`, `closed_at`, `settled_at`.
- Unique: `(pair_id, session_date)`.

**trust_ledger**
- `id`, `session_id fk`, `deal_id fk`, `side` (a_to_b|b_to_a), `token`, `network`, `amount_token`, `amount_usd`, `type` (deal|adjustment|fee), `created_at`.
- Index: `(session_id, created_at)`.

**eod_settlement**
- `id`, `session_id fk`, `payer_company_id fk`, `payee_company_id fk`, `token`, `network`, `amount_token`, `amount_usd`, `outgoing_tx_id fk NULL`, `status` (queued|sent|confirmed|partial|failed), `created_at`.

---

## 8) Споры и арбитраж
**dispute**
- `id`, `deal_id fk NULL`, `trust_session_id fk NULL`, `opened_by_company_id fk`, `against_company_id fk`, `status` (open|under_review|resolved|closed), `type` (deal|trust), `reason_code text`, `claim_text`, `created_at`, `resolved_at NULL`.

**dispute_evidence**
- `id`, `dispute_id fk`, `by_company_id fk`, `type` (tx_hash|bank_doc|screenshot|video|note|other), `url_or_hash`, `meta jsonb`, `created_at`.

**dispute_decision**
- `id`, `dispute_id fk`, `decision` (refund|penalty|split|reject), `amount_token`, `amount_usd`, `beneficiary_company_id fk`, `rationale_text`, `decided_by_member_id fk`, `created_at`.

---

## 9) Интеграции и уведомления
**api_key**
- `id`, `company_id fk`, `name`, `token_hash`, `scopes text[]`, `ip_allowlist text[]`, `created_at`, `revoked_at NULL`.

**webhook_endpoint**
- `id`, `company_id fk`, `url`, `events text[]`, `secret`, `is_active bool`, `created_at`.

**notification** (Inbox в @ClubGateBot)
- `id`, `member_id fk`, `type` (offer_new|deal_update|payout|deposit|dispute|trust_pair|trust_eod), `payload jsonb`, `is_read bool`, `created_at`.

---

## 10) Админка и аудит
**risk_policy_change**
- `id`, `changed_by_member_id fk`, `what jsonb`, `created_at`.

**audit_log**
- `id`, `actor_member_id fk`, `action` (string), `entity` (table), `entity_id uuid`, `diff jsonb`, `created_at`.

---

## 11) Бизнес‑правила и ограничения
- `deal.amount_token` ≤ `company_limit_snapshot.deal_cap_usd` и ≤ `network.tx_cap_usd` (для auto‑выплат).
- `mode=deposit` перед `in_progress`: проверка достаточности депозита; запись `deposit_ledger(type='hold')`.
- **Trust‑пары:** разрешены только если обе компании имеют `deposit.status='active'`. На пару действует `daily_credit_limit_usd`; при 90% — автопауза новых неттинговых сделок.
- **EOD:** планировщик по TZ пары закрывает `trust_session` в `cutoff_local_time`, считает `net`, создаёт `eod_settlement` и выплаты (батчинг по `network.tx_cap_usd`).
- Вывод депозита допустим только при `open_exposure_usd=0`, отсутствии споров и `trust_reserve`=0 (или после `trust_release`).
- Все выплаты требуют 2FA и, при превышении порога tier, подтверждения Approver.

---

## 12) Индексы (минимальный набор)
- `offer(status, direction, token_id, network_id)` — лента.
- `deal(initiator_company_id, state)`, `deal(counterparty_company_id, state)` — рабочие очереди.
- `incoming_tx(to_address, status)` и `incoming_tx(deal_id)` — авто‑матчинг/выплаты.
- `deposit_ledger(company_id, created_at)` — выписки.
- `trust_pair(company_low_id, company_high_id)` и `trust_session(pair_id, session_date)` — неттинг.
- `trust_ledger(session_id, created_at)` — расчёт EOD.
- `notification(member_id, is_read, created_at)` — Inbox.

---

## 13) Представления (views)
- `v_company_limits` — депозит, k, cap’ы, экспозиция, trust_reserve.
- `v_deal_timeline` — сводный таймлайн по `deal_event` + платежам.
- `v_trust_session_summary` — нетто по парам за день, статусы выплат.
- `v_dispute_summary` — открытые споры, суммы, сроки.

---

## 14) Миграция v1 → v2 (эскиз)
1. Добавить таблицы `trust_pair`, `trust_session`, `trust_ledger`, `eod_settlement`.
2. В `deal` — поля `is_trusted_netting`, `trust_session_id` (NULLABLE).
3. В `deposit_ledger` — типы `trust_reserve`/`trust_release`.
4. В `exposure_snapshot` — поле `trust_reserve_usd`.
5. Создать индексы, представления.
6. Бэкфилл `timezone` и `work_hours` для компаний (дефолт: Europe/Warsaw).

---
