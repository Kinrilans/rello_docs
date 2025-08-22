# API‑контракты (v1)

Версия: v1 · Формат: JSON/HTTPS · База URL: `https://api.example.com/v1`\
Аудитория: интеграции партнёров и внутренних ботов. Токены: USDT/USDC. Сети: ERC‑20/TRC‑20.

---

## 0) Общие правила

**Аутентификация:** `Authorization: Bearer <api_key>` (см. `POST /api-keys`). Для админ‑ботов — отдельные ключи/роли.\
**Scopes:** `companies:read`, `offers:write`, `deals:write`, `treasury:write`, `trust:write`, `arbiter:write`, и т.д.\
**2FA для критичных операций:** header `X-2FA-Code: <TOTP>` (вывод депозита, pay\_out, крупные лимиты).\
**Идемпотентность:** для всех `POST`/`PUT` денежных действий — `Idempotency-Key: <uuid>` (365 дней).\
**Пагинация:** cursor‑based: `?limit=50&cursor=...` → `{items:[...], next_cursor:"..."}`.\
**Сортировка/фильтры:** query‑параметры; даты в ISO 8601 (UTC).\
**Ошибки (единый формат):** `{ "error": { "code": "string", "message": "human text", "details": {...}, "trace_id": "uuid" } }`\
**Версионирование:** префикс `/v1`; минорные флаги — через Header `X-Features: a,b`.\
**Рейт‑лимиты:** по умолчанию 10 rps / 100 burst / ключ. Заголовки: `X-RateLimit-*`.

---

## 1) Deeplink/JWT для ботов

**Назначение:** безопасное переключение между ботами Telegram.\
**Формат:** Telegram `start=<jwt>`; JWT RS256, TTL 2–5 мин.\
**Пейлоад:**

```json
{
  "sub": "member_id",
  "company_id": "uuid",
  "role": ["trader","company_admin"],
  "partner_id": "uuid",
  "deal_id": "uuid",
  "trust_session_id": "uuid",
  "iat": 1734890000,
  "exp": 1734890120
}
```

**Заголовок:** `kid` для ротации ключей.\
**Проверки:** валидность подписи, срок, статус участника, доступ к сущности.

---

## 2) Компании и пользователи

### POST /auth/token (опц., внутреннее)

Выдать short‑lived токен боту по api\_key (machine‑to‑machine).

### GET /companies

Фильтры: `status`, `tier`, `q`. Ответ: список компаний.

### GET /companies/{company\_id}

Вернёт профиль, tier, лимиты (snapshot), активные доверительные пары (id/лимит/cutoff/TZ).

### GET /members?company\_id=...

Список участников компании (id, роли, 2FA).

---

## 3) Реквизиты (шаблоны)

### GET /bank-accounts?company\_id=...

### POST /bank-accounts

Тело: `{ currency, account_name, iban_or_number, bank_name, swift_bic, routing, label, is_default }`

### GET /wallets?company\_id=...

### POST /wallets

Тело: `{ network:"ERC20|TRC20", token:"USDT|USDC", address, memo_tag, label }`

### POST /wallets/{id}/verify

Подтвердить владение (подпись/микро‑tx). Тело: `{ method:"sign|micro_tx", proof:{...} }`

---

## 4) Офферы

### GET /offers

Фильтры: `status`, `direction:cash_in|cash_out`, `network`, `token`, `mode:pass_through|deposit`, `company_id`, `limit/cursor`.

### POST /offers

```json
{
  "company_id":"...",
  "direction":"cash_in",
  "mode":"deposit",
  "network":"TRC20",
  "token":"USDT",
  "fiat_currency":"EUR",
  "amount_min":1000,
  "amount_max":50000,
  "price_type":"markup",
  "markup_bps":45,
  "deadline_hours":24,
  "trusted_netting_default": true
}
```

Ответ: `offer`.

### POST /offers/{offer\_id}/accept

Создаёт `deal (proposed)`; требует `Idempotency-Key`.

---

## 5) Сделки

### GET /deals

Фильтры: `state`, `company_id`, `counterparty_id`, `network`, `token`, `created_from/to`, `mode`, `is_trusted_netting`.

### GET /deals/{deal\_id}

Карточка сделки + таймлайн событий.

### PATCH /deals/{deal\_id}

Обновить дедлайн/примечания/флаги. Тело: `{ deadline_at, notes, trusted_netting: true|false }`

### POST /deals/{deal\_id}/confirm

Подтвердить действие стороны. Тело:

```json
{ "type":"fiat_sent|fiat_received|crypto_sent|crypto_received", "evidence": {"tx_hash":"...","files":["url1"]} }
```

### POST /deals/{deal\_id}/cancel

Требует основание; может требовать согласия второй стороны.

### POST /deals/{deal\_id}/dispute

Создаёт спор. Тело: `{ reason_code, claim_text, attachments:[...]} `

---

## 6) Treasury: депозиты и транзакции

### GET /deposits?company\_id=...

Баланс/статус/адреса.

### POST /deposits/withdraw

Тело: `{ company_id, network, token, to_address, amount_token }`\
Заголовки: `Idempotency-Key`, `X-2FA-Code`. Ответ: `withdraw_request`.

### GET /incoming-tx?deal\_id=... | /incoming-tx/{tx\_id}

Входящие на кошельки платформы; статусы: `seen|confirmed|failed`.

### POST /pay-in

Привязать входящую к сделке (pass‑through). Тело: `{ deal_id, network, token, expected_amount_token }`

### POST /pay-out

Разовая выплата из корпоративного баланса. Тело: `{ company_id, network, token, to_address, amount_token, purpose }`\
Заголовки: `Idempotency-Key`, `X-2FA-Code`. Ответ: `outgoing_tx`.

### GET /payouts

Фильтры: период/сеть/статус/инициатор/тип (`pass_through|manual|eod`).

---

## 7) Trust: пары, сессии, ledger, EOD

### GET /trust/pairs?company\_id=...

Список пар.

### POST /trust/pairs

```json
{
  "company_a_id":"...",
  "company_b_id":"...",
  "tokens":["USDT","USDC"],
  "networks":["ERC20","TRC20"],
  "daily_credit_limit_usd":60000,
  "reserve_pct":15,
  "timezone":"Europe/Warsaw",
  "cutoff_local_time":"18:00"
}
```

Статус: `proposed` → активируется после согласия B (см. `POST /trust/pairs/{id}/accept`).

### POST /trust/pairs/{id}/accept | /pause | /resume | /update

Изменения действуют со **следующей** дневной сессии.

### GET /trust/sessions?pair\_id=...&date=YYYY-MM-DD

Получить дневную сессию пары.

### GET /trust/sessions/{session\_id}/ledger

Построчный ledger: сделки/adjustments/fees.

### POST /trust/sessions/{session\_id}/ledger

Добавить корректировку (арбитр/система): `{ type:"adjustment", side:"a_to_b|b_to_a", token, network, amount_token, note }`

### POST /trust/sessions/{session\_id}/close

Принудительно закрыть сессию в cut‑off (админ/система). Считает net и создаёт `eod_settlement`.

### GET /trust/settlements?date=...

Список EOD‑расчётов дня (статусы: `queued|sent|confirmed|partial|failed`).

### POST /trust/settlements/{id}/recompute

Пересчитать net после корректировок ledger.

---

## 8) Споры (Arbiter)

### GET /disputes?type=deal|trust&status=...

Очередь споров.

### GET /disputes/{id}

Карточка спора; ссылки на `deal`, `trust_session`, материалы.

### POST /disputes

Создать спор: `{ type:"deal|trust", deal_id?, trust_session_id?, reason_code, claim_text, attachments:[...] }`

### POST /disputes/{id}/evidence

Добавить материалы: `{ type:"tx_hash|bank_doc|screenshot|video|note", url_or_hash, meta }`

### POST /disputes/{id}/resolve

Вынести решение (2FA арбитра):

```json
{
  "decision":"refund|penalty|split|reject",
  "amount_token": 1000,
  "token":"USDT",
  "beneficiary_company_id":"...",
  "rationale_text":"..."
}
```

Побочный эффект: записи в `deposit_ledger` и (при необходимости) создание `outgoing_tx`.

---

## 9) Webhooks и события

### Управление endpoint’ами

- `GET /webhooks` — список.
- `POST /webhooks` — `{ url, events:[...], secret, ip_allowlist[] }`
- `DELETE /webhooks/{id}`

### Подпись

Headers: `X-Signature: sha256=<hex>`, `X-Timestamp`, `X-Event-Id`. Формат: `HMAC_SHA256(secret, timestamp + body)`.

### События (перечень)

- **offers:** `offer.created`, `offer.updated`, `offer.closed`
- **deals:** `deal.created`, `deal.updated`, `deal.auto_payout_sent`, `deal.completed`, `deal.disputed`
- **treasury/deposits:** `deposit.funded`, `deposit.withdraw.requested`, `deposit.withdraw.sent`, `payout.sent`
- **trust:** `trust.pair.activated`, `trust.pair.paused`, `trust.session.closed`, `trust.ledger.adjusted`, `trust.eod_settlement.sent|confirmed|partial|failed`
- **arbiter:** `dispute.opened`, `arbiter.decision.posted`

### Пример payload

```json
{
  "event":"trust.eod_settlement.sent",
  "id":"evt_123",
  "created_at":"2025-08-22T18:00:05Z",
  "data":{
    "settlement_id":"...",
    "pair_id":"...",
    "session_id":"...",
    "payer_company_id":"...",
    "payee_company_id":"...",
    "network":"TRC20",
    "token":"USDT",
    "amount_token": 12500,
    "tx_hash":"..."
  }
}
```

---

## 10) Примеры последовательностей

### A) Pass‑through

1. `POST /offers` → `POST /offers/{id}/accept` → `POST /pay-in` → `deal.auto_payout_sent` webhook.

### B) Deposit‑mode

1. `POST /offers` (mode=deposit) → `accept` → подтверждения `fiat_sent/crypto_sent` → `resolve/complete`.

### C) Trust + EOD

1. `POST /trust/pairs` → `/accept` → сделки с `trusted_netting=true` → `POST /trust/sessions/{id}/close` (или авто) → `trust.eod_settlement.sent` → `...confirmed`.

---

## 11) Справочник enum’ов (фрагмент)

- `direction`: `cash_in|cash_out`
- `mode`: `pass_through|deposit`
- `deal.state`: `proposed|funded|in_progress|completed|cancelled|disputed|resolved`
- `dispute.status`: `open|under_review|resolved|closed`
- `network`: `ERC20|TRC20`
- `token`: `USDT|USDC`
- `trust.session.state`: `open|closed|settled|disputed`
- `eod_settlement.status`: `queued|sent|confirmed|partial|failed`

---

## 12) Маппинг на БД (кратко)

- `offer` ↔ таблица `offer`
- `deal`/`confirm` ↔ `deal`, `deal_event`
- `pay-in/out` ↔ `incoming_tx`, `outgoing_tx`
- `deposits`/`withdraw` ↔ `deposit`, `deposit_ledger`
- `trust/*` ↔ `trust_pair`, `trust_session`, `trust_ledger`, `eod_settlement`
- `disputes/*` ↔ `dispute`, `dispute_evidence`, `dispute_decision`
- `webhooks/*` ↔ `webhook_endpoint`

---

## 13) Примечания безопасности

- Все вебхуки — с повторной доставкой и `X-Event-Id` (идемпотентность на стороне приёмника).
- Для денежных вызовов — обязательные `Idempotency-Key` и, при необходимости, `X-2FA-Code`.
- Deeplink/JWT — только RS256, TTL ≤ 5 минут, один‑разовый use (server‑side nonce).
- Адреса выплат — только из allow‑list компании; добавление адресов — через верификацию в шлюзе.

