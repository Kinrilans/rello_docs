# Deeplink/JWT (v1) — безопасные переходы между ботами

Цель: безопасно передавать контекст (company/role/partner/deal/trust\_session) между ботами Telegram (@ClubGateBot → @DealDeskBot/@TreasuryBot/@ArbiterDeskBot) без раскрытия чувствительных данных и с защитой от повторного использования.

---

## 0) Термины и носители

**Deeplink** — ссылка вида `https://t.me/<BotName>?start=<token>` (для приватного чата).\
**Носитель токена:**

1. **Short‑Token (рекомендуется)** — короткий непрозрачный идентификатор (≈22–32 b64url символа), который ссылается на серверный «тикет» с полным контекстом. Легко помещается в deeplink и выдерживает ограничения Telegram.
2. **JWT (JWS Compact)** — подписанный токен RS256/ES256 с минифицированными полями. Удобен для независимой проверки, но длиннее. Используем **только при гарантии длины** (или как вложение в WebApp).

Обе схемы обладают одинаковой семантикой (одноразовые, короткоживущие). В этой версии поддерживаем **оба варианта**.

---

## 1) Модель безопасности

- **TTL:** 2–5 минут по умолчанию (`exp`), можно короче для повышенного риска.
- **Одноразовость:** `jti`/ticket\_id кэшируется как «использован» на TTL+5 минут (replay‑защита).
- **Аудитория:** `aud = "bot:<name>"` (например, `bot:DealDesk`). Подписанный токен действителен только для указанного бота.
- **Привязка к Telegram:** проверяем соответствие **sender.tg\_user\_id** входящего апдейта с claim `tg_id` (или с owner тикета). Несовпадение → отказ.
- **Роли и доступ:** сверяем `company_id`, активность участника, роли (`roles[]`), статус компании (не suspended), наличие 2FA, если требуются.
- **Тонкие права (scope):** включаем минимально необходимое, например: `scope:["open:deal","offer:read","trust:read"]`. Скоуп действует только в рамках начальной операции.

---

## 2) Структура токена

### A) Short‑Token (серверный тикет)

- Генерация: криптослучайный 128–192 bit → **base64url без паддинга** (≈22–32 символа).
- Таблица `deeplink_ticket`:
  - `id` (token), `payload jsonb`, `aud`, `ttl_sec`, `single_use` (true), `created_at`, `used_at NULL`, `used_by_tg_id NULL`, `ip_user_agent_hash` (опц.).
- Пример `payload`:

```json
{
  "sub": "member_uuid",
  "company_id": "uuid",
  "roles": ["trader","company_admin"],
  "tg_id": 123456789,
  "scope": ["open:deal","offers:read"],
  "ctx": {
    "partner_id": "uuid",
    "deal_id": "uuid",
    "trust_session_id": "uuid"
  },
  "nbf": 1734890000,
  "exp": 1734890120
}
```

### B) JWT (JWS Compact)

- Алгоритм: **RS256** (или **ES256** для компактности). Заголовок: `{ "alg":"RS256", "kid":"<key-id>", "typ":"JWT" }`.
- Поля (минимизируем ключи):

```json
{
  "iss":"rello-platform",
  "aud":"bot:DealDesk",
  "sub":"member_uuid",
  "cid":"company_uuid",
  "rol":["trader"],
  "tg":123456789,
  "scp":["open:deal"],
  "ctx":{"pid":"partner_uuid","did":"deal_uuid","tsid":"trust_session_uuid"},
  "jti":"uuid",
  "iat":1734890000,
  "nbf":1734890000,
  "exp":1734890120
}
```

- Сжатие: при необходимости допускается **DEF‑компрессия** полезной нагрузки перед подписью (флаг `zip:"DEF"` в заголовке JWS).

---

## 3) Жизненный цикл

1. **Mint**: исходный бот (или бэкенд) вызывает внутренний API `POST /deeplink/tickets` с `aud`, `payload`, `ttl_sec`, `single_use=true` → получает `{ token, url }` для конкретного бота.
2. **Deliver**: бот отправляет ссылку/кнопку пользователю.
3. **Consume**: целевой бот на `/start <token>`:
   - валидирует токен (подпись для JWT / existence для short‑token),
   - сверяет `aud`, `exp/nbf`, `jti/used_at`, `tg_id`, `roles`, `scope`, статус компании,
   - помечает `used_at` (и `jti` в кэше),
   - поднимает **контекст сессии** и выполняет стартовую операцию (открыть сделку, профиль пары, экран EOD и т. п.).
4. **Expire**: истёкшие/использованные токены удаляются задачей‑пылесосом.

---

## 4) Маппинг прав (scope → действие)

| Scope                       | Что разрешено при входе                                              |
| --------------------------- | -------------------------------------------------------------------- |
| `offers:read`               | Открыть ленту офферов с предустановленными фильтрами                 |
| `open:deal`                 | Открыть карточку конкретной сделки/создать тред (если есть контекст) |
| `trust:read`                | Открыть карточку доверительной пары или дневной сессии               |
| `treasury:settlements:read` | Открыть экран EOD‑расчётов (фильтр по паре/дате из `ctx`)            |
| `arbiter:case:read`         | Открыть карточку спора                                               |

Scopes действуют **только в момент входа**; дальнейшие действия проходят обычные ACL/роли.

---

## 5) Примеры deeplink‑переходов

1. **@ClubGateBot → @DealDeskBot** открыть сделку `deal_id`:

```
https://t.me/DealDeskBot?start=<token>
// aud=bot:DealDesk, scp=[open:deal], ctx={"did":"..."}
```

2. **@ClubGateBot → @TreasuryBot** на вкладку сегодняшних EOD по паре:

```
https://t.me/TreasuryBot?start=<token>
// aud=bot:Treasury, scp=[treasury:settlements:read,trust:read], ctx={"pid":"...","tsid":"..."}
```

3. **@DealDeskBot → @ArbiterDeskBot** в конкретный спор:

```
https://t.me/ArbiterDeskBot?start=<token>
// aud=bot:Arbiter, scp=[arbiter:case:read], ctx={"did":"...","case":"..."}
```

---

## 6) Проверка токена (серверная логика)

**Общее:**

1. Извлечь `token` из `start`.
2. Если токен по виду **opaque** → найти в `deeplink_ticket` по `id` и сверить поля/сроки/одноразовость.
3. Если токен — **JWT** → проверить JWS (по `kid` найти ключ, проверить `iss/aud/nbf/exp/jti`).
4. Сверить `tg_id` с реальным Telegram user id апдейта; сверить `company_id`, `roles`, `scope`.
5. Отметить использованным (`used_at` / кэш `jti`).
6. Сформировать контекст сессии бота и выполнить действие.

**Псевдокод (JWT‑ветка):**

```python
jwt = parse(token)
assert jwt.header.alg in ["RS256","ES256"]
key = jwks.get(jwt.header.kid)
claims = verify_and_decode(jwt, key)
now = time()
assert claims['iss'] == 'rello-platform'
assert claims['aud'] == expected_aud
assert claims['nbf'] <= now < claims['exp']
assert not replay_cache.exists(claims['jti'])
assert claims['tg'] == telegram_update.from_user.id
assert role_check(claims['cid'], claims['rol'], claims['scp'])
replay_cache.put(claims['jti'], ttl=claims['exp']-now+300)
```

---

## 7) Управление ключами (для JWT)

- Хранение ключей: `JWKS` (JSON Web Key Set) с активными публичными ключами; `kid` в заголовке JWS.
- Ротация: поддерживаем **два активных ключа** (старый/новый) в течение окна; приватные ключи в HSM/secure vault.
- Отзыв: при инциденте можно мгновенно перевести всех на Short‑Token и отключить приём JWT до замены ключей.

---

## 8) Ошибки и коды отказа

- `token_invalid` — подпись неверна/токен битый.
- `token_expired` — вышел TTL.
- `token_replay` — уже использован.
- `aud_mismatch` — не тот бот.
- `role_denied` — нет прав/роль не подходит.
- `company_suspended` — компания выключена.
- `tg_mismatch` — токен не принадлежит этому Telegram‑пользователю.

Ответ пользователю: краткое human‑сообщение + кнопка «Вернуться в @ClubGateBot».

---

## 9) Лимиты и телеметрия

- Rate‑limit на выдачу токенов: по умолчанию 5 rps на компанию (всплески — через квоты).
- Метрики: количество выданных/успешных/ошибочных переходов; топ причин отказов; средний TTL; доля JWT vs Short‑Token.

---

## 10) Советы по длине токена

- Для deeplink рекомендуется **Short‑Token** (короткий, не содержит PII). JWT используем преимущественно для внутренних редиректов и WebApp, либо минимизируем поля (`rol` вместо `roles`, `cid` вместо `company_id` и т. п.) и, при необходимости, включаем `zip:"DEF"`.

---

## 11) Совместимость

- Оба носителя (Short‑Token/JWT) поддерживаются одинаково; выбор задаётся флагом на стороне сервера. Клиентские боты не зависят от формата — они только получают `start`.

