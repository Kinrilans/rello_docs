# События и вебхуки (v1) — полная спецификация

Версия: v1 · Доставка: **at‑least‑once** · Формат: JSON/HTTPS · Подпись: HMAC‑SHA256\
Сферы: offers, deals, treasury (deposits/payouts), trust (pairs/sessions/EOD), arbiter (disputes/decisions).

---

## 0) Модель доставки

- **At‑least‑once**: событие может прийти повторно. Получатель обязан делать **дедуп** по `id`.
- **Порядок**: внутри одного агрегата (например, `deal_id`) сохраняется **best‑effort** порядок; глобально порядок не гарантируется.
- **Семантика ACK**: любой **2xx** в ответе = принято; всё остальное — ошибка → ретрай.
- **Таймауты**: 10 сек на установление соединения, 20 сек на ответ.
- **Ограничения**: тело ≤ **256 KB**. Для больших вложений — ссылки на pre‑signed файлы.

---

## 1) Управление endpoint’ами

`POST /v1/webhooks` → `{ url, events:["deal.*","trust.eod_settlement.*"], secret, ip_allowlist[] }`\
`GET /v1/webhooks` → список\
`DELETE /v1/webhooks/{id}` → отключение

**Фильтрация**: поддерживается `*` по префиксу (напр., `deal.*`, `trust.*`).

---

## 2) Безопасность и подпись

**Заголовки** (в каждом запросе):

- `X-Webhook-Id`: UUID события доставки (не путать с `event.id`).
- `X-Event-Id`: UUID самого события (уникален глобально).
- `X-Event-Type`: напр., `deal.created`.
- `X-Event-Version`: целое, напр., `1`.
- `X-Timestamp`: UNIX‑time (секунды).
- `X-Attempt`: номер попытки (1..N).
- `X-Signature`: `sha256=<hex>`.

**Подпись**: `payload_to_sign = X-Timestamp + "." + raw_body`\
`signature = HMAC_SHA256(secret, payload_to_sign)` → присылается в `X-Signature`.

**Проверки у получателя**:

1. `abs(now - X-Timestamp) ≤ 5 мин` (защита от replay).
2. Пересчитать HMAC и сравнить в режиме **constant‑time**.
3. Проверить, что `X-Event-Id` не обрабатывался (кэш 24ч).
4. Обработать **ровно один раз** (идемпотентность на вашей стороне).

**Ротация секрета**: поддерживаются **два активных секрета** (старый/новый) на переходный период. Сверять по обоим.

---

## 3) Ретрай‑политика

- **Бэкофф с джиттером**: \~0с → 30с → 2м → 10м → 30м → 1ч → 3ч … (до **24 часов** максимум).
- Сетевые ошибки/таймаут = ошибка. HTTP≥500 = ошибка. HTTP 429 → уважать `Retry-After`.
- После 24ч неудач событие помечается `dead_letter` (доступно для ручного реплея в админке).

---

## 4) Реплей событий

- `GET /v1/events?since=2025-08-22T00:00:00Z&types=deal.*,trust.*&limit=100` → история (только для владельца endpoint’а).
- `POST /v1/webhooks/{id}/replay` → `{ from:"2025-08-22T00:00:00Z", to?:"...", types?:["..."] }`.

---

## 5) Формат события

```json
{
  "id": "evt_5f4e...",
  "type": "trust.eod_settlement.sent",
  "version": 1,
  "created_at": "2025-08-22T16:00:05Z",
  "source": "platform",
  "data": { /* см. схемы ниже */ }
}
```

**Замечания**: `id` — уникален; `type` стабильный; `version` увеличивается при несовместимых изменениях; временные поля — UTC ISO8601.

---

## 6) JSON‑схемы `data` (фрагменты)

### 6.1 deals

``

```json
{
  "deal_id": "uuid",
  "offer_id": "uuid",
  "initiator_company_id": "uuid",
  "counterparty_company_id": "uuid",
  "mode": "pass_through|deposit",
  "network": "ERC20|TRC20",
  "token": "USDT|USDC",
  "amount_token": "12345.67",
  "deadline_at": "2025-08-22T18:00:00Z",
  "is_trusted_netting": false
}
```

``

```json
{
  "deal_id": "uuid",
  "network": "TRC20",
  "token": "USDT",
  "amount_token": "10000",
  "tx_hash": "0x...",
  "payout_wallet_label": "hot-1"
}
```

### 6.2 treasury

``

```json
{
  "company_id": "uuid",
  "deposit_id": "uuid",
  "network": "ERC20",
  "token": "USDT",
  "amount_token": "50000",
  "tx_hash": "0x...",
  "confirmations": 6
}
```

``

```json
{
  "company_id": "uuid",
  "outgoing_tx_id": "uuid",
  "network": "TRC20",
  "token": "USDT",
  "amount_token": "250000",
  "tx_hash": "...",
  "purpose": "manual|pass_through|eod"
}
```

### 6.3 trust

``

```json
{
  "pair_id": "uuid",
  "company_a_id": "uuid",
  "company_b_id": "uuid",
  "timezone": "Europe/Warsaw",
  "cutoff_local_time": "18:00",
  "daily_credit_limit_usd": 60000,
  "reserve_pct": 15
}
```

``

```json
{
  "session_id": "uuid",
  "pair_id": "uuid",
  "session_date": "2025-08-22",
  "net_token": "USDT",
  "net_amount_token": "12500",
  "payer_company_id": "uuid",
  "payee_company_id": "uuid"
}
```

``

```json
{
  "settlement_id": "uuid",
  "session_id": "uuid",
  "pair_id": "uuid",
  "network": "TRC20",
  "token": "USDT",
  "amount_token": "12500",
  "tx_hash": "...",
  "status": "sent|confirmed|partial|failed",
  "partial_remaining_token": "0"  
}
```

### 6.4 arbiter

``

```json
{
  "dispute_id": "uuid",
  "type": "deal|trust",
  "deal_id": "uuid",
  "trust_session_id": "uuid",
  "opened_by_company_id": "uuid",
  "reason_code": "string"
}
```

``

```json
{
  "dispute_id": "uuid",
  "decision": "refund|penalty|split|reject",
  "amount_token": "5000",
  "token": "USDT",
  "beneficiary_company_id": "uuid"
}
```

---

## 7) Примеры подписи (псевдокод)

```python
# server side verify
raw_body = request.get_data(as_text=False)
ts = request.headers['X-Timestamp']
msg = ts.encode() + b'.' + raw_body
sig = hmac_sha256(secret, msg).hex()
secure_compare(sig, request.headers['X-Signature'].split('=')[1])
```

---

## 8) Отказоустойчивость и масштабирование

- Очередь доставки — отдельная на **каждый endpoint** (изоляция подписчиков).
- Конкурентность: до **5** параллельных попыток на endpoint (управляется).
- Dead‑letter queue для событий с неудачной доставкой >24ч; ручной реплей из админки.
- Мониторинг: метрики по попыткам/латентности/ошибкам; алерты при всплесках.

---

## 9) Требования к получателю

- Держать **allow‑list IP** отправителя (опционально).
- Отвечать **2xx** только после успешной **идемпотентной** записи (например, в БД).
- Хранить обработанные `X-Event-Id` ≥ 24ч.
- Учитывать, что порядок событий может отличаться (сверять даты и состояния по API при сомнениях).

---

## 10) Диагностика

- Заголовки ответа получателя логируются.
- В каждом запросе передаётся `X-Attempt` и `X-Webhook-Id` для трассировки.
- В админке доступен **replay** по `event.id`/времени/типу.

---

## 11) Совместимость и версии

- При добавлении полей — **minor** (обратная совместимость).
- При изменении типов/семантики — **version++** в `X-Event-Version` и в поле `version` JSON.
- Получатели должны **игнорировать незнакомые поля**.

