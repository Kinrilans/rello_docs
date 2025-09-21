# Rello Club — API‑клиенты и безопасность (MVP v1.0)

**Статус:** Черновик (согласовано)

Документ описывает единый подход к аутентификации, подписи запросов (HMAC), защите от повторов (anti‑replay), идемпотентности, ротации ключей, rate‑лимитам, логированию и безопасной интеграции всех ботов с финансовым ядром.

---

## 1) Область действия
- **Клиенты:** @RelloOfficeBot, @RelloAdminBot, @RelloClubBot, @RelloDealBot, @RelloGroupBot, @RelloTopicsBot, @RelloNoticeBot — у каждого **свой API‑ключ и набор скоупов**.
- **Окружения:** `dev` и `prod` (раздельные ключи, БД и URL).
- **IP‑allowlist:** `127.0.0.1` и приватная подсеть VDS (например, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`).

---

## 2) Заголовки и общие требования
| Заголовок | Обязательно | Назначение |
|---|---|---|
| `X-Api-Key` | всегда | Идентификатор клиента (без секрета). |
| `X-Signature` | всегда | Подпись **HMAC‑SHA256 (base64)** канонической строки. |
| `X-Timestamp` | всегда | ISO 8601 UTC, например `2025-09-21T12:00:00Z`. Допустимое окно ± **300 с**. |
| `X-Idempotency-Key` | для небезопасных методов | Уникальный ключ идемпотентности (TTL 24 часа). |
| `X-Correlation-Id` | всегда | Трассировка запроса (при отсутствии ядро сгенерирует). |
| `Content-Type` | для тела | `application/json; charset=utf-8`. |

**Транспорт:** только **HTTPS**. В `prod` — публичный сертификат (например, Let’s Encrypt), в `dev` — допускается self‑signed.

**Proxy:** доверяем заголовкам `X-Forwarded-*` **только** от нашего reverse‑proxy.

---

## 3) Подпись (HMAC‑SHA256)

### 3.1 Каноническая строка
```
<HTTP_METHOD_UPPER>\n
<PATH>\n
<CANONICAL_QUERY>\n
<SHA256_HEX_OF_BODY>\n
<X-Timestamp>\n
<X-Idempotency-Key_or_empty>
```
**Правила:**
- `PATH` — абсолютный путь (например, `/v1/rc/topups`), **без** домена.
- `CANONICAL_QUERY` — параметры запроса, отсортированные по ключам (и по значениям при повторяющихся ключах), кодировка RFC 3986, формат `k=v` через `&`. Если параметров нет — пустая строка.
- `SHA256_HEX_OF_BODY` — для пустого тела используется хеш от пустой строки.
- Последняя строка — `X-Idempotency-Key` или пусто, если заголовок не используется.

**Подпись:** `signature = base64( HMAC_SHA256(secret, canonical_string) )`.

### 3.2 Anti‑replay
- Ядро отклоняет запросы, если `| now − X-Timestamp | > 300 с`.
- Подпись привязана к времени и (при наличии) `X-Idempotency-Key` ⇒ повторить позже **нельзя** без новой подписи.

### 3.3 Пример (тест‑вектор)
- Метод: `POST`
- Путь: `/v1/rc/topups`
- Query: *(пусто)*
- Тело (JSON, компактно): `{"amount_rc":"100.000000","owner_id":"11111111-1111-1111-1111-111111111111"}`
- SHA256(body): `be17500f5a1162129498d5a3cb7338a868a4855be5b9e8c27bae0bddcc669d87`
- `X-Timestamp`: `2025-09-21T12:00:00Z`
- `X-Idempotency-Key`: `idemp-12345`
- Секрет: `test_secret_ABC123`
- Каноническая строка:
```
POST
/v1/rc/topups

be17500f5a1162129498d5a3cb7338a868a4855be5b9e8c27bae0bddcc669d87
2025-09-21T12:00:00Z
idemp-12345
```
- `X-Signature`: `zn7Dl+jrzFWyZASXUVqR/GgZ+GGKwHa6fgqd/hfwTZc=`

---

## 4) Идемпотентность (24 часа)
- Применяется ко всем **небезопасным** операциям ядра (например: создание пополнения/вывода RC, старт сделки/холда, заявки депозита, выпуск токена группы и т.п.).
- TTL ключа — **24 часа**. Ключ уникален в рамках `(client, endpoint)`.
- Повтор запроса с **тем же** телом → возврат ранее сохранённого ответа.
- Повтор с **иным** телом → `409 RC-STATE-3001 IdempotencyConflict`.

---

## 5) Корреляция
- Клиент передаёт `X-Correlation-Id` (UUID или trace‑ID). Ядро возвращает его во всех ответах и пишет в аудит.
- При отсутствии заголовка ядро сгенерирует значение автоматически.

---

## 6) Скоупы и роли ключей
Примеры скоупов (минимальный набор):
- `wallet:read`, `wallet:write`, `deals:*`, `deposit:*`, `group:*`, `ads:*`, `notice:send`, `admin:*`.

Рекомендации:
- **OfficeBot** — `admin:*` (суперпользователь).
- **AdminBot** — `moderation:*`, `arbitration:*`, `deposit:*`, `wallet:manual`.
- **ClubBot** — `company:*`, `group:*`, `deals:*`, `wallet:read|write` (только свои сущности).
- **DealBot** — `deals:*`, `files:*` (ведение сделок).
- **GroupBot** — `group:*`, `deals:*`.
- **TopicsBot** — `ads:*`.
- **NoticeBot** — `notice:send`.

> Итоговые права ограничиваются **и** скоупами ключа, **и** ролью пользователя на уровне сущностей (см. `roles_permissions_matrix.md`).

---

## 7) Ротация и отзыв ключей
- У ключа есть поле `rotates_after` (момент плановой ротации).
- **Мягкая ротация:** 14 дней — старый и новый ключи активны параллельно.
- **Отзыв:** немедленный; ядро возвращает `401 RC-AUTH-1003 ApiKeyRevoked`.

---

## 8) Лимиты и размеры
- **Rate‑limits (per api_key):** 120 запросов/мин; допустимые всплески — до 20 rps на 10 секунд. Превышение: `429 RC-AUTH-1010 RateLimited` + `Retry-After`.
- **Body‑size:** максимум **256 KB**. Крупные файлы передаются через Telegram/файловое хранилище, в API — ссылки (`tg_file_id`/URL).

---

## 9) Ответы и ошибки
- Ошибки строго по `error_catalog.md` (код, категория, `user_message`/`developer_message`, `params`, `details`).
- Локаль ошибки по умолчанию задаётся **ключом клиента** (например, OfficeBot — `ru`, остальные — `en`), но может переопределяться `Accept-Language`.

---

## 10) Примеры (Node.js)

### 10.1 Подпись запроса
```js
const crypto = require('crypto');

function sha256Hex(str) {
  return crypto.createHash('sha256').update(str, 'utf8').digest('hex');
}

function canonicalQuery(params) {
  if (!params || Object.keys(params).length === 0) return '';
  const enc = (s) => encodeURIComponent(s).replace(/[!'()*]/g, c => '%' + c.charCodeAt(0).toString(16).toUpperCase());
  const pairs = [];
  for (const k of Object.keys(params).sort()) {
    const vals = [].concat(params[k]).map(String).sort();
    for (const v of vals) pairs.push(`${enc(k)}=${enc(v)}`);
  }
  return pairs.join('&');
}

function canonicalString({ method, path, query, body, timestamp, idempotencyKey }) {
  const bodyStr = body ? JSON.stringify(body) : '';
  const bodyHash = sha256Hex(bodyStr);
  const queryStr = canonicalQuery(query);
  return [
    method.toUpperCase(),
    path,
    queryStr,
    bodyHash,
    timestamp,
    idempotencyKey || ''
  ].join('\n');
}

function sign(secret, canonical) {
  return crypto.createHmac('sha256', Buffer.from(secret, 'utf8')).update(canonical, 'utf8').digest('base64');
}

// usage
const method = 'POST';
const path = '/v1/rc/topups';
const query = {};
const body = { amount_rc: '100.000000', owner_id: '11111111-1111-1111-1111-111111111111' };
const timestamp = new Date().toISOString();
const idempotencyKey = 'idemp-12345';
const canonical = canonicalString({ method, path, query, body, timestamp, idempotencyKey });
const signature = sign(process.env.API_SECRET, canonical);

const headers = {
  'X-Api-Key': process.env.API_KEY,
  'X-Signature': signature,
  'X-Timestamp': timestamp,
  'X-Idempotency-Key': idempotencyKey,
  'X-Correlation-Id': crypto.randomUUID(),
  'Content-Type': 'application/json; charset=utf-8'
};
```

### 10.2 Проверка на стороне ядра (псевдокод)
```js
// 1) Проверить IP в allowlist
// 2) Проверить X-Timestamp (|now - ts| <= 300s)
// 3) Реконструировать canonicalString из фактических path/query/body
// 4) Вычислить HMAC и сравнить constant‑time с X-Signature
// 5) Проверить скоупы по X-Api-Key
// 6) Идемпотентность: если ключ уже встречался — вернуть сохранённый ответ / 409 при конфликте
```

---

## 11) Интеграция ядро → NoticeBot
- Выделенный ключ NoticeBot; тот же HMAC‑процесс.
- Дедупликация по `notice_id` на стороне NoticeBot; ядро гарантирует уникальность.
- SLA и форматы — см. `notice_contract.md`.

---

## 12) Логи, аудит, PII
- В аудит пишем: `correlation_id`, `api_key_id`, метод, путь, код ответа, длительность.
- Маскируем ончейн‑адреса и личные данные (первые 4/последние 4 символа).
- Срок хранения логов: **30 дней**. Доступ — только главному админу (SSH).

---

## 13) Требования по методам
| Метод | Тело | `X-Idempotency-Key` | Примечание |
|---|---:|:---:|---|
| GET | нет | опционально | Подпись включает хеш пустого тела. |
| DELETE | нет | опционально | Только на безопасных эндпоинтах. |
| POST | да | **обязателен** | Для всех операций создания/проведения. |
| PUT/PATCH | да | **обязателен** | Для всех операций изменения состояния. |

---

## 14) Частые ошибки (справочно)
- `RC-AUTH-1001 InvalidSignature` — некорректная подпись/канонизация.
- `RC-AUTH-1002 ClockSkew` — вышли за окно 300 с.
- `RC-AUTH-1010 RateLimited` — превышение лимитов (уважать `Retry-After`).
- `RC-PERM-1101 RoleMissing` — у ключа нет нужного скоупа.
- `RC-STATE-3001 IdempotencyConflict` — повтор с иным телом.
- `RC-PROVIDER-3402 ProviderUnavailable` — временные сбои провайдера.

---

## 15) Управление ключами и секретами (процедуры)
**Выдача ключа:**
1. Создать клиента в ядре: `api_clients.insert(name, scopes, allowed_ips)`.
2. Сгенерировать `api_key` и `secret` (≥ 32 байт, криптографически стойкие).
3. Передать команде бота **только** по защищённому каналу (SSH/1Password/Vault).
4. На стороне бота сохранить в Vault или `.env` (только dev); права файла: `0400`.

**Мягкая ротация:**
1. Создать новый ключ с теми же скоупами/IP.
2. Перевести бота на **двойную конфигурацию** (оба ключа активны).
3. Наблюдать **14 дней**.
4. Отозвать старый ключ (`revoked`), удалить из конфигурации.

**Отзыв (инцидент/утечка):**
1. Немедленно `revoked` → ядро отвечает `401 RC-AUTH-1003`.
2. Аудит логов по `api_key_id` + `correlation_id`.
3. Выпустить новый ключ с принципом наименьших привилегий.
4. Обновить конфигурации, при необходимости очистить кеш идемпотентности.

---

## 16) Подключение нового бота (чек‑лист)
1. Зарегистрировать клиента → получить `api_key`/`secret` и `scopes`.
2. Добавить исходящий IP бота в allowlist.
3. Включить HMAC‑подпись; заголовки: `X-Api-Key`, `X-Signature`, `X-Timestamp`, `X-Correlation-Id`, для POST/PUT — `X-Idempotency-Key`.
4. Проверить синхронизацию времени (NTP): `| now − X-Timestamp | ≤ 300 с`.
5. Прогнать smoke‑тесты (см. §17): `GET /health`, POST с идемпотентностью, ошибки 401/403/409/429.
6. Включить нужные разделы/возможности на стороне бота.

---

## 17) Тест‑кейсы интеграции (минимум)
- ✅ Подпись верна → 200.
- ❌ Неверная подпись → 401 `RC-AUTH-1001`.
- ❌ Просроченный `X-Timestamp` (> 300 с) → 401 `RC-AUTH-1002`.
- ✅ Идемпотентность: повтор с тем же body → 200 (идентичный ответ).
- ❌ Идемпотентность: повтор с иным body → 409 `RC-STATE-3001`.
- ❌ Нет скоупа → 403 `RC-PERM-1101`.
- ❌ Превышение лимита → 429 `RC-AUTH-1010` + `Retry-After`.

---

## 18) Примеры запросов (cURL)
### 18.1 GET с query
```
curl -sS -G "https://api.rello.example/v1/wallets" \
  --data-urlencode "owner_id=11111111-1111-1111-1111-111111111111" \
  -H "X-Api-Key: <API_KEY>" \
  -H "X-Timestamp: 2025-09-21T12:00:00Z" \
  -H "X-Correlation-Id: <UUID>" \
  -H "X-Signature: <base64(hmac)>"
```

### 18.2 POST — создание вывода RC (с идемпотентностью)
```
curl -sS -X POST "https://api.rello.example/v1/rc/withdrawals" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "X-Api-Key: <API_KEY>" \
  -H "X-Timestamp: 2025-09-21T12:00:00Z" \
  -H "X-Idempotency-Key: idemp-12345" \
  -H "X-Correlation-Id: <UUID>" \
  -H "X-Signature: <base64(hmac)>" \
  -d '{"owner_id":"11111111-1111-1111-1111-111111111111","amount_rc":"150.000000","prefer_token":"USDT","destination_address":"T...abcd"}'
```

### 18.3 Ядро → NoticeBot (отправка уведомления)
```
curl -sS -X POST "https://notice.rello.example/v1/notice" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "X-Api-Key: <NOTICE_KEY>" \
  -H "X-Timestamp: 2025-09-21T12:00:00Z" \
  -H "X-Correlation-Id: <UUID>" \
  -H "X-Signature: <base64(hmac)>" \
  -d '{"notice_id":"...","type":"deal.created","recipients":[{"owner_id":"..."}],"payload":{"deal_id":"..."}}'
```

---

## 19) Скоупы → действия (ориентир)
| Скоуп | Разрешённые действия (пример) |
|---|---|
| `wallet:read` | Просмотр балансов, истории леджера своих сущностей |
| `wallet:write` | Инициировать topup/withdraw RC, выпуск адреса пополнения |
| `deals:*` | Создание/ведение сделок, холд/переводы, арбитраж (клиентский уровень) |
| `group:*` | Привязка групп, изменение комиссий группы, просмотр статистики |
| `ads:*` | Создание объявлений (Topics) |
| `notice:send` | Отправка конвертов в NoticeBot |
| `deposit:*` | Создание/обработка заявок депозита (только для админов) |
| `admin:*` | Глобальные настройки (OfficeBot), управление администраторами |

---

## 20) Pre‑prod чек‑лист безопасности
- [ ] Все боты используют HTTPS и актуальные корневые сертификаты.
- [ ] Включён IP‑allowlist для всех ключей.
- [ ] HMAC‑подпись проверяется на всех эндпоинтах.
- [ ] `X-Timestamp` проверяется с окном 300 с; NTP синхронизирован.
- [ ] Идемпотентность включена на POST/PUT; TTL 24 часа.
- [ ] Rate‑лимит настроен: 120 req/min per key; алерты при всплесках 429.
- [ ] Секреты хранятся в Vault/1Password; доступ ограничен.
- [ ] Логи маскируют адреса/PII; срок хранения 30 дней.
- [ ] Ротация ключей проверена на стенде (двойная конфигурация 14 дней).

---

## 21) Изменения и версия
**MVP v1.0**
- Введены: HMAC‑подпись, anti‑replay (±300 с), идемпотентность (24 часа), rate‑лимиты, IP‑allowlist.
- Раздельные ключи/скоупы для каждого бота.
- Процедуры ротации/отзыва; примеры Node.js и cURL.

**Следующие версии (план):**
- mTLS внутри приватной сети (mutual auth).
- JTI‑реестр для дополнительной защиты `X-Idempotency-Key` за пределами TTL.
- Политики `scopes by endpoint` в OpenAPI‑контракте.
