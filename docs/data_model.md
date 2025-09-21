# Rello Club — Модель данных (MVP v1.0)

**Статус:** Черновик (согласованная база)

Документ описывает **логическую модель данных** для финансового ядра и поддерживающих ботов. Содержит перечень сущностей, таблиц, полей, инвариантов и индексов, необходимых для реализации MVP. Имена полей/таблиц указаны ориентировочно; допустимы косметические переименования при имплементации.

---

## 0) Условности

- **ID:** UUID v4 (`id`); внешние ссылки (Telegram и пр.) — в полях `ext_*`.
- **Время:** `created_at`, `updated_at` (`timestamptz`, UTC).
- **Валюты/точность:** `RC`, `USDT`, `USDC`, `EUR`.
  - `RC`: `numeric(38,6)`
  - `USDT/USDC` (TRC20): `numeric(38,6)`
  - `EUR`: `numeric(38,2)`
- **Округление:** round half up (банковское).
- **Soft‑delete:** только для нефинансовых справочников (`deleted_at timestamptz`). Финансовые записи — append‑only.
- **Регистрация владельцев (Вариант B):** боты регистрируют владельца в ядре; ядро возвращает канонический `owner_id`, который используется во всех операциях.

---

## 1) Идентичность и владение

### 1.1 `owners`

Канонические субъекты, к которым привязаны балансы, сделки, депозиты.

- `id` (UUID, PK)
- `kind` (enum: `user`, `group`, `company`, `service`)
- `ext_ref` (text, опц.) — внешняя ссылка (напр., Telegram ID/ID группы)
- `label` (text) — человекочитаемое имя (компания/группа)
- `created_at`, `updated_at`

**Инвариант:** владелец `service` — единственный (платформа).

### 1.2 `users`

Справочник пользователей (для аудита/UI).

- `id` (UUID, PK)
- `owner_id` (FK → owners.id, kind=`user`)
- `ext_telegram_id` (text)
- `display_name` (text), `locale` (text)
- `created_at`, `updated_at`

### 1.3 `companies`

Компании клиентов (тенанты) на платформе; могут иметь работников и группы.

- `id` (UUID, PK)
- `owner_id` (FK → owners.id, kind=`company`) — канонический владелец средств
- `client_owner_user_id` (FK → owners.id, kind=`user`) — кто управляет этой компанией
- `name` (text)
- `default_locale` (text, `en`|`ru`) — язык компании по умолчанию для всех ботов; настраивается владельцем в @RelloClubBot.
- `created_at`, `updated_at`, `deleted_at`

### 1.4 `groups`

Телеграм‑группы, привязанные к компании, со своим RC‑кошельком.

- `id` (UUID, PK)
- `owner_id` (FK → owners.id, kind=`group`)
- `company_id` (FK → companies.id)
- `ext_telegram_group_id` (text)
- `title` (text)
- `group_locale` (text, `en`|`ru`) — язык этой чат-группы; по умолчанию наследуется от `company.default_locale`, может меняться командой в группе с @RelloGroupBot.
- `created_at`, `updated_at`, `deleted_at`

### 1.5 `workers`

Связь пользователей с компаниями/группами и их роль.

- `id` (UUID, PK)
- `user_owner_id` (FK → owners.id, kind=`user`)
- `company_id` (FK → companies.id)
- `group_id` (FK → groups.id, nullable)
- `role` (enum: `worker`, `manager`)
- `created_at`, `updated_at`, `deleted_at`

---

## 2) Кошельки и леджер (RC)

### 2.1 `rc_wallets`

Один RC‑кошелёк на `owner`. Поле баланса — **кэш**; источником истины служит журнал проводок.

- `id` (UUID, PK)
- `owner_id` (FK → owners.id)
- `balance_rc` (numeric(38,6), default 0)
- `locked_rc` (numeric(38,6), default 0) — активные холды
- `created_at`, `updated_at`

### 2.2 Леджер (double‑entry)

**План счетов** (справочник): `ledger_accounts`

- `id` (UUID, PK)
- `code` (text, unique), `title` (text)
- `class` (enum: `asset`, `liability`, `income`, `expense`, `equity`)
- `currency` (enum: `RC`, `USDT`, `USDC`, `EUR`)
- `owner_id` (nullable FK → owners.id) — для субсчетов конкретного владельца (rc‑кошелёк, эскроу)
- `purpose` (enum: `rc_wallet`, `escrow_hold`, `service_fee`, `client_fee`, `deposit`, `service_main`, `service_dirty`)
- `created_at`

**Заголовки проводок:** `ledger_journal`

- `id` (UUID, PK)
- `correlation_id` (text) — из `X-Correlation-Id`
- `kind` (enum: `rc_topup`, `rc_withdraw`, `deal_hold`, `deal_success`, `deal_cancel`, `arbitrage_win`, `arbitrage_split`, `arbitrage_refund`, `fee_charge`, `deposit_topup`, `deposit_withdraw`)
- `created_at`

**Строки проводок:** `ledger_entries`

- `id` (UUID, PK)
- `journal_id` (FK → ledger\_journal.id)
- `account_id` (FK → ledger\_accounts.id)
- `amount` (numeric(38,6)) — **положительное** число; сторона задаётся по правилу для счёта
- `direction` (enum: `debit`, `credit`)
- `meta` (jsonb) — снимки: проценты комиссий, курсы, ссылки на сделки/tx

**Инвариант:** сумма дебетов = сумма кредитов в рамках одного `journal_id`.

### 2.3 `escrow_holds`

Холды для сделок (RC заморожены до исхода).

- `id` (UUID, PK)
- `deal_id` (FK → deals.id)
- `owner_id` (FK → owners.id) — чьи RC заморожены
- `amount_rc` (numeric(38,6))
- `status` (enum: `active`, `released`, `transferred`, `refunded`)
- `created_at`, `updated_at`

---

## 3) Интеграция с блокчейном (TRC20)

### 3.1 Сервисные кошельки (внешние)

`service_external_wallets`

- `id` (UUID, PK)
- `role` (enum: `main`, `dirty`)
- `token` (enum: `USDT`, `USDC`)
- `network` (enum: `TRC20`)
- `address` (text, unique)
- `created_at`

### 3.2 Пополнения (TRC20 → RC)

`topups`

- `id` (UUID, PK)
- `owner_id` (FK → owners.id)
- `token` (enum: `USDT`, `USDC`)
- `network` (enum: `TRC20`)
- `deposit_address` (text)
- `reusable_until_funded` (bool, default true)
- `status` (enum: `pending`, `funded`, `aml_checking`, `aml_high_risk`, `credited`, `expired`, `refunded`)
- `expires_at` (timestamptz) — обычно +24ч
- `tx_in_hash` (text), `tx_in_url` (text)
- `credited_rc_amount` (numeric(38,6))
- `aml_report_id` (FK → aml\_reports.id, nullable)
- `created_at`, `updated_at`

### 3.3 Выводы (RC → TRC20)

`withdrawals`

- `id` (UUID, PK)
- `owner_id` (FK → owners.id)
- `amount_rc` (numeric(38,6))
- `prefer_token` (enum: `USDT`, `USDC`)
- `effective_token` (enum: `USDT`, `USDC`)
- `network` (enum: `TRC20`)
- `destination_address` (text)
- `status` (enum: `pending`, `aml_checking`, `sending`, `sent`, `failed`)
- `aml_report_id` (FK → aml\_reports.id, nullable)
- `tx_out_hash` (text), `tx_out_url` (text)
- `created_at`, `updated_at`

### 3.4 AML‑отчёты

`aml_reports`

- `id` (UUID, PK)
- `provider` (enum: `GetBlock`)
- `direction` (enum: `in`, `out`)
- `address` (text)
- `token` (enum: `USDT`, `USDC`)
- `network` (enum: `TRC20`)
- `risk` (enum: `low`, `medium`, `high`, `unknown`)
- `report_ref` (text) — ссылка провайдера
- `details` (jsonb)
- `created_at`
- **Ретеншн:** ≥ 12 месяцев (операционная политика)

---

## 4) Сделки и арбитраж

### 4.1 `deals`

- `id` (UUID, PK)
- `type` (enum: `B2C`, `C2B`)
- `status` (enum: `draft`, `held`, `success`, `cancelled`, `arbitration`)
- `amount_rc` (numeric(38,6))
- `payer_owner_id` (FK → owners.id)
- `payee_owner_id` (FK → owners.id)
- `hold_side` (enum: `payer`, `payee`) — у кого холд
- `ad_id` (text, nullable) — если из Topics
- `created_at`, `updated_at`

### 4.2 `deal_commissions_snapshot`

- `deal_id` (PK/FK → deals.id)
- `client_fee_percent` (numeric(9,4))
- `service_fee_percent` (numeric(9,4))
- `total_client_fee_rc` (numeric(38,6))
- `total_service_fee_rc` (numeric(38,6))

### 4.3 `deal_arbitration`

- `deal_id` (PK/FK → deals.id)
- `outcome` (enum: `win`, `split`, `refund`)
- `admin_owner_id` (FK → owners.id, kind=`user`)
- `reason` (text)
- `created_at`

### 4.4 Файлы/артефакты (ретеншн 30 дней)

`deal_files`

- `id` (UUID, PK)
- `deal_id` (FK → deals.id)
- `kind` (enum: `check`, `doc`, `message_export`)
- `tg_file_id` (text), `url` (text, nullable)
- `retention_until` (timestamptz) — по умолчанию +30 дней
- `created_at`

---

## 5) Комиссии и конфигурация

### 5.1 `service_fees`

Версионируемые глобальные комиссии (источник — OfficeBot).

- `id` (UUID, PK)
- `valid_from` (timestamptz)
- `rc_topup_fee_percent` (numeric(9,4))
- `rc_withdraw_fee_percent` (numeric(9,4)) — используется и как комиссия возврата при AML high‑risk
- `deposit_topup_fee_percent` (numeric(9,4))
- `deposit_withdraw_fee_percent` (numeric(9,4))
- `deal_service_fee_percent` (numeric(9,4))
- `created_by_owner_id` (FK → owners.id)
- `created_at`

### 5.2 `group_client_fees`

Дефолтные комиссии продавца для сделок в группе; с историей изменений.

- `id` (UUID, PK)
- `group_id` (FK → groups.id)
- `valid_from` (timestamptz)
- `b2c_percent` (numeric(9,4))
- `c2b_percent` (numeric(9,4))
- `created_by_owner_id` (FK → owners.id)
- `created_at`

### 5.3 `config_params`

Параметры платформы с историей.

- `id` (UUID, PK)
- `key` (text, уникально для активной записи)
- `value` (text)
- `valid_from` (timestamptz)
- `created_by_owner_id` (FK → owners.id)
- `created_at`

**Примеры ключей:** `rc_topup_min`, `group_token_ttl_hours`, `token_rate_limit_per_min`, `after_restore_lock_hours`.

---

## 6) Депозиты (USDT TRC20, ручные операции)

Депозиты хранятся **вне ядра** (холодные кошельки). Ядро ведёт учёт балансов и заявок на **ручную** обработку модераторами в @RelloAdminBot.

### 6.1 `deposit_accounts`

- `id` (UUID, PK)
- `owner_id` (FK → owners.id)
- `token` (enum: `USDT`) — MVP
- `network` (enum: `TRC20`)
- `ext_cold_address` (text) — справочно (без ончейн‑автоматики)
- `balance_token` (numeric(38,6)) — зеркальный баланс
- `updated_at`

### 6.2 `deposit_requests`

- `id` (UUID, PK)
- `owner_id` (FK → owners.id)
- `kind` (enum: `topup`, `withdraw`)
- `amount_token` (numeric(38,6))
- `status` (enum: `pending`, `approved`, `rejected`, `executed`)
- `moderator_owner_id` (FK → owners.id, nullable)
- `comment` (text)
- `created_at`, `updated_at`

**Примечание:** комиссии по операциям депозита берутся из `service_fees` на момент операции и отражаются в `ledger_*`.

---

## 7) Реферальная программа

Вознаграждения считаются из **прибыли сервиса** по сделкам рефералов; уровни зависят от **собственных** завершённых сделок реферера.

### 7.1 `ref_program_config`

Версионируемый конфиг.

- `id` (UUID, PK)
- `valid_from` (timestamptz)
- `base_rate` (numeric(9,4)) — напр., 0.0500 (5%)
- `max_rate` (numeric(9,4)) — напр., 0.1000 (10%)
- `tiers` (jsonb) — список `{min_own_deals, rate}`
- `created_by_owner_id`
- `created_at`

### 7.2 `ref_relations`

- `referrer_owner_id` (FK → owners.id)
- `referee_owner_id` (FK → owners.id)
- `created_at`
- **PK:** (`referrer_owner_id`, `referee_owner_id`)

### 7.3 `ref_monthly_accruals`

- `id` (UUID, PK)
- `referrer_owner_id` (FK → owners.id)
- `month` (date)
- `rate_applied` (numeric(9,4))
- `accrued_rc` (numeric(38,6)) — из прибыли сервиса по сделкам рефералов
- `own_deals_completed` (integer) — снимок для расчёта уровня
- `created_at`, `updated_at`

### 7.4 `ref_payouts`

- `id` (UUID, PK)
- `referrer_owner_id` (FK → owners.id)
- `month` (date)
- `amount_rc` (numeric(38,6))
- `status` (enum: `scheduled`, `paid`, `failed`)
- `tx_ref` (text, nullable) — внутренняя ссылка
- `created_at`, `updated_at`

---

## 8) API‑клиенты, идемпотентность и outbox

### 8.1 `api_clients`

- `id` (UUID, PK)
- `name` (text) — имя бота/сервиса
- `api_key` (text, unique)
- `secret_hash` (text)
- `is_active` (bool)
- `allowed_ips` (text[])
- `rotates_after` (timestamptz)
- `created_at`, `updated_at`

### 8.2 `idempotency_keys`

- `key` (text, PK)
- `endpoint` (text)
- `request_hash` (bytea)
- `response_snapshot` (jsonb)
- `owner_client_id` (FK → api\_clients.id)
- `created_at`, `expires_at`

### 8.3 `notice_outbox`

Локальный дедуп и надёжная доставка в NoticeBot.

- `id` (UUID, PK)
- `notice_id` (text, unique)
- `kind` (text)
- `payload` (jsonb)
- `status` (enum: `pending`, `sent`, `failed`)
- `last_error` (text)
- `created_at`, `updated_at`

---

## 9) Аудит и доступ

### 9.1 `audit_log`

- `id` (UUID, PK)
- `actor_owner_id` (FK → owners.id)
- `action` (text)
- `target_table` (text), `target_id` (text)
- `before` (jsonb), `after` (jsonb)
- `correlation_id` (text)
- `created_at`

### 9.2 Security/Locks

`security_locks`, `security_bans` (отражают семантику API)

- `security_locks`: `id`, `user_owner_id`, `type` (enum: `post_restore_24h`), `created_at`, `released_at`
- `security_bans`: `user_owner_id` (PK), `reason`, `created_at`, `released_at`

---

## 10) Индексы и ограничения (неполный список)

- `owners(kind, ext_ref)` — быстрый поиск по внешним ссылкам.
- `rc_wallets(owner_id)` — уникальный кошелёк на владельца.
- `ledger_entries(journal_id)`; `ledger_entries(account_id)`; **контроль** дебет=кредит в транзакции.
- `escrow_holds(deal_id, status)`.
- `topups(owner_id, status, expires_at)`; `topups(deposit_address)` — уникально при активности.
- `withdrawals(owner_id, status)`.
- `aml_reports(address, token, network, created_at)`.
- `deals(status, created_at)`; `deal_commissions_snapshot(deal_id)` — уникально.
- `group_client_fees(group_id, valid_from)`; `service_fees(valid_from)`.
- `config_params(key, valid_from)`.
- `ref_relations(referrer_owner_id, referee_owner_id)` — уникально; `ref_monthly_accruals(referrer_owner_id, month)` — уникально.
- `api_clients(api_key)` — уникально; `idempotency_keys(key)` — уникально; `notice_outbox(notice_id)` — уникально.

---

## 11) Инварианты и бизнес‑правила (сводка)

- **RC ≡ USDT** для конверсии; все конверсии EUR опираются на спот EUR/USDT.
- **Частичных закрытий нет**; холд — на всю сумму.
- **AML IN:** high‑risk → возврат минус `rc_withdraw_fee_percent`; остаток в `dirty`‑кошелёк.
- **AML OUT:** high‑risk адрес назначения → отклонить вывод.
- **Блок после восстановления:** 24 ч запрет операций пользователя.
- **Хранение файлов:** 30 дней (файлы сделок), AML‑отчёты ≥ 12 месяцев.
- **Депозиты:** заявки обрабатываются вручную; балансы синхронизируются; холодные адреса — справочные.

---

## 12) Открытые вопросы (трек)

- Расширение на другие сети/токены (в будущем): добавить `token` шире в леджер/счета.
- Детализация структуры прибыли сервиса сверх `total_service_fee_rc` (сети, провайдеры и т.д.).
- Переопределения комиссий на уровне компании (если потребуется позже).

---

## 13) Эскиз ERD (логический, упрощённый)

```
owners (user|group|company|service) --< rc_wallets
owners --< deals >-- owners
          |              |
          |              >-- escrow_holds
          >-- topups -- aml_reports
          >-- withdrawals -- aml_reports

companies --< groups --< group_client_fees
users --< workers >-- companies

service_fees (versioned)    config_params (versioned)
ledger_accounts --< ledger_entries >-- ledger_journal

ref_program_config --< ref_monthly_accruals >-- owners(referrer)
owners(referrer) --< ref_relations >-- owners(referee)

api_clients --< idempotency_keys
notice_outbox
audit_log
```

