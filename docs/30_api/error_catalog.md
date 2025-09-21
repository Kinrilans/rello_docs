# Rello Club — Каталог ошибок (MVP v1.0)

**Статус:** Черновик (согласовано)

Единый справочник кодов ошибок для финансового ядра и ботов. Используется:

- в ответах REST API ядра;
- в сообщениях @RelloNoticeBot (см. `notice_contract.md`).

---

## 1) Формат и состав ошибки

### 1.1 Код и категории

- **Код:** `RC-<CATEGORY>-<NNNN>` (пример: `RC-AUTH-1001`).
- **Категории:**
  - `AUTH` — ключи/подпись/время
  - `PERM` — права/баны
  - `INPUT` — валидация/входные данные
  - `STATE` — неверный статус/переход/идемпотентность
  - `FUNDS` — баланс/ликвидность/локи
  - `AML` — проверки AML
  - `RATE` — курсы/конвертация
  - `PROVIDER` — внешние провайдеры
  - `NOTICE` — доставка уведомлений
  - `SECURITY` — ограничения безопасности (24ч-блок)
  - `REFERRAL` — реферальная программа
  - `SYSTEM` — внутренние/неизвестные

### 1.2 Схема ответа ошибки (REST)

```json
{
  "code": "RC-AML-3201",
  "http_status": 403,
  "category": "AML",
  "user_message": "Вывод отклонён: адрес назначения помечен как высокий риск.",
  "user_locale": "ru",
  "developer_message": "Destination address high risk per AML provider.",
  "params": {"destination_address": "Txxx…abcd"},
  "details": {"aml": {"summary": "Mixer exposure", "reason_codes": ["MIXER","SCAM"], "report_ref": "GB-abc123"}},
  "correlation_id": "d0d3…",
  "retry_after": 0,
  "can_retry": false,
  "ts": "2025-09-21T12:34:56Z"
}
```

**Правила:**

- `user_message` локализуется (EN/RU); при отсутствии локализации — **EN**.
- `developer_message` — всегда EN, только для логов/отладки.
- В `user_message` и `params` **маскируем** адреса/идентификаторы (`Txxx…abcd`). Полные значения — только в `developer_message`/логах.
- Для AML-ошибок включать `details.aml` (summary/reason\_codes/report\_ref).

### 1.3 Маппинг на HTTP

| Категория                               | Тип                        | HTTP    |
| --------------------------------------- | -------------------------- | ------- |
| AUTH                                    | подпись/ключ/время         | **401** |
| PERM / SECURITY                         | нет прав/бан/24ч-блок      | **403** |
| INPUT.not\_found                        | ресурс не найден           | **404** |
| STATE / IDEMPOTENCY                     | конфликт состояния         | **409** |
| INPUT.business                          | бизнес-валидация           | **422** |
| RATE\_LIMIT                             | частые запросы             | **429** |
| PROVIDER.gateway                        | ошибки провайдера          | **502** |
| PROVIDER.unavailable / RATE.unavailable | провайдер/курсы недоступны | **503** |
| SYSTEM                                  | иное                       | **500** |

---

## 2) Диапазоны кодов

- `AUTH` **1000–1099**
- `PERM` **1100–1199**
- `INPUT` **2000–2099**
- `STATE` **3000–3099**
- `FUNDS` **3100–3199**
- `AML` **3200–3299**
- `RATE` **3300–3399**
- `PROVIDER` **3400–3499**
- `NOTICE` **3500–3599**
- `SECURITY` **3600–3699**
- `REFERRAL` **3700–3799**
- `SYSTEM` **9000–9099**

---

## 3) Справочник (минимально необходимый набор)

### AUTH (1000–1099)

| Код          | Имя              | HTTP | user\_message (RU)                            | user\_message (EN)                           |
| ------------ | ---------------- | ---- | --------------------------------------------- | -------------------------------------------- |
| RC-AUTH-1001 | InvalidSignature | 401  | Неверная подпись запроса.                     | Invalid request signature.                   |
| RC-AUTH-1002 | ClockSkew        | 401  | Несовпадение времени клиента. Проверьте часы. | Client clock skew detected. Check your time. |
| RC-AUTH-1003 | ApiKeyRevoked    | 401  | Ключ API отозван.                             | API key revoked.                             |
| RC-AUTH-1010 | RateLimited      | 429  | Слишком много запросов. Повторите позже.      | Too many requests. Try again later.          |

### PERM (1100–1199)

| Код          | Имя         | HTTP | RU                                           | EN                                        |
| ------------ | ----------- | ---- | -------------------------------------------- | ----------------------------------------- |
| RC-PERM-1100 | BannedUser  | 403  | Доступ запрещён (пользователь заблокирован). | Access denied (user banned).              |
| RC-PERM-1101 | RoleMissing | 403  | Нет прав для выполнения операции.            | Missing required role for this operation. |

### INPUT (2000–2099)

| Код           | Имя                  | HTTP | RU                                         | EN                      |
| ------------- | -------------------- | ---- | ------------------------------------------ | ----------------------- |
| RC-INPUT-2000 | ValidationFailed     | 422  | Данные не прошли проверку.                 | Validation failed.      |
| RC-INPUT-2001 | CurrencyNotSupported | 422  | Валюта не поддерживается.                  | Currency not supported. |
| RC-INPUT-2002 | AddressInvalid       | 422  | Некорректный адрес.                        | Invalid address.        |
| RC-INPUT-2003 | AmountTooLow         | 422  | Сумма ниже минимально допустимой.          | Amount below minimum.   |
| RC-INPUT-2004 | NotFound             | 404  | Объект не найден.                          | Resource not found.     |
| RC-INPUT-2007 | TooMuchData          | 422  | Слишком большой объём данных для экспорта. | Export too large.       |

### STATE (3000–3099)

| Код           | Имя                   | HTTP | RU                                              | EN                                  | Примечание                                                                   |
| ------------- | --------------------- | ---- | ----------------------------------------------- | ----------------------------------- | ---------------------------------------------------------------------------- |
| RC-STATE-3000 | DealStateInvalid      | 409  | Недопустимое состояние сделки.                  | Invalid deal state.                 |                                                                              |
| RC-STATE-3001 | IdempotencyConflict   | 409  | Конфликт идемпотентности (повтор с иным телом). | Idempotency conflict.               |                                                                              |
| RC-STATE-3002 | HoldMissing           | 409  | Холда не существует или уже снят.               | Hold missing or released.           |                                                                              |
| RC-STATE-3003 | AlreadyProcessed      | 409  | Операция уже выполнена.                         | Operation already processed.        |                                                                              |
| RC-STATE-3004 | TransitionNotAllowed  | 409  | Переход состояния запрещён.                     | State transition not allowed.       |                                                                              |
| RC-STATE-3005 | ManualModeDisabled    | 409  | Ручной режим отключён администратором.          | Manual mode disabled by admin.      | **Применяется только к новым заявкам, созданным после переключения режима.** |
| RC-STATE-3006 | DepositAutoNotAllowed | 409  | Депозит обрабатывается только вручную.          | Deposit operations are manual-only. | В системе нет авто-режима депозитов.                                         |

### FUNDS (3100–3199)

| Код           | Имя                   | HTTP | RU                                     | EN                                    |
| ------------- | --------------------- | ---- | -------------------------------------- | ------------------------------------- |
| RC-FUNDS-3100 | InsufficientRcBalance | 422  | Недостаточно средств RC на балансе.    | Insufficient RC balance.              |
| RC-FUNDS-3101 | InsufficientLiquidity | 503  | Недостаточно ликвидности для операции. | Insufficient liquidity for operation. |

### AML (3200–3299)

| Код         | Имя                 | HTTP | RU                                                              | EN                                                       |
| ----------- | ------------------- | ---- | --------------------------------------------------------------- | -------------------------------------------------------- |
| RC-AML-3200 | IncomingHighRisk    | 403  | Поступление с адреса высокого риска. Средства будут возвращены. | Incoming from high‑risk address. Funds will be refunded. |
| RC-AML-3201 | DestinationHighRisk | 403  | Адрес назначения высокий риск. Укажите другой адрес.            | Destination address high risk. Provide another address.  |
| RC-AML-3202 | AmlProviderTimeout  | 503  | Провайдер AML недоступен. Повторите позже.                      | AML provider timeout/unavailable.                        |

### RATE (3300–3399)

| Код          | Имя               | HTTP | RU                               | EN                   |
| ------------ | ----------------- | ---- | -------------------------------- | -------------------- |
| RC-RATE-3300 | FxRateUnavailable | 503  | Курс недоступен.                 | FX rate unavailable. |
| RC-RATE-3301 | PairUnsupported   | 422  | Валютная пара не поддерживается. | Pair not supported.  |

### PROVIDER (3400–3499)

| Код              | Имя                 | HTTP | RU                            | EN                             |
| ---------------- | ------------------- | ---- | ----------------------------- | ------------------------------ |
| RC-PROVIDER-3400 | CryptoApiTimeout    | 502  | Крипто‑провайдер не отвечает. | Crypto API timeout.            |
| RC-PROVIDER-3401 | CryptoApiError      | 502  | Ошибка крипто‑провайдера.     | Crypto API error.              |
| RC-PROVIDER-3402 | ProviderUnavailable | 503  | Внешний провайдер недоступен. | External provider unavailable. |

### NOTICE (3500–3599)

| Код            | Имя                    | HTTP | RU                                                 | EN                                   |
| -------------- | ---------------------- | ---- | -------------------------------------------------- | ------------------------------------ |
| RC-NOTICE-3500 | RecipientUndeliverable | 200  | Получатель недоступен в Telegram (не нажал Start). | Recipient undeliverable in Telegram. |

### SECURITY (3600–3699)

| Код              | Имя                   | HTTP | RU                                                           | EN                                                |
| ---------------- | --------------------- | ---- | ------------------------------------------------------------ | ------------------------------------------------- |
| RC-SECURITY-3600 | PostRestoreLockActive | 403  | Операции заблокированы на 24ч после восстановления аккаунта. | Operations locked for 24h after account recovery. |

### REFERRAL (3700–3799)

| Код              | Имя               | HTTP | RU                                          | EN                            |
| ---------------- | ----------------- | ---- | ------------------------------------------- | ----------------------------- |
| RC-REFERRAL-3700 | TierConfigMissing | 422  | Конфигурация уровней рефералки отсутствует. | Referral tier config missing. |

### SYSTEM (9000–9099)

| Код            | Имя             | HTTP | RU                     | EN                |
| -------------- | --------------- | ---- | ---------------------- | ----------------- |
| RC-SYSTEM-9000 | UnexpectedError | 500  | Непредвиденная ошибка. | Unexpected error. |

---

## 4) Рекомендации по обработке на стороне ботов/клиентов

- **Показывайте **`` и действия (CTA), основанные на коде. Примеры:
  - `RC-AML-3201` → кнопка «Изменить адрес вывода» с диплинком в RelloDealBot.
  - `RC-FUNDS-3100` → «Пополнить кошелёк» (deeplink в ClubBot).
  - `RC-STATE-3005` → «Создать новую заявку (авто‑режим активен)».
- **Не показывайте **`` пользователю; логируйте вместе с `correlation_id`.
- При `RateLimited` (1010) уважайте `Retry-After` из заголовков/поля.

---

## 5) Связь с уведомлениями (Notice)

- События `*.failed`, `*.aml.high`, `ban.*` **обязаны** включать `error_code` из данного каталога.
- В Notice также действуют правила локализации (см. `notice_contract.md`).

---

## 6) Изменения режимов обработки

- Переключатель **ручной/авто** для операций **RC** задаётся в @RelloOfficeBot главным админом.
- Изменение режима **распространяется только на новые заявки**, созданные после переключения, чтобы избежать путаницы с уже активными.
- Для **депозитов** авто‑режима **не существует**: любые попытки автоматизации должны отвечать `RC-STATE-3006`.

---

## 7) Примеры

### 7.1 Ошибка AML при выводе (RU)

```json
{
  "code": "RC-AML-3201",
  "http_status": 403,
  "category": "AML",
  "user_message": "Вывод отклонён: адрес назначения помечен как высокий риск.",
  "params": {"destination_address": "TQ1a…9ZkP"},
  "details": {"aml": {"summary": "Mixer exposure", "reason_codes": ["MIXER"], "report_ref": "GB-xyz"}},
  "correlation_id": "fe91…",
  "can_retry": false
}
```

### 7.2 Конфликт идемпотентности (EN)

```json
{
  "code": "RC-STATE-3001",
  "http_status": 409,
  "category": "STATE",
  "user_message": "Request already submitted with different body.",
  "developer_message": "Idempotency key reuse with different payload.",
  "correlation_id": "2c77…",
  "can_retry": false
}
```

