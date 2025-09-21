# Rello Club — Контракт уведомлений (Notice) (MVP v1.0)

**Статус:** Черновик (согласовано с владельцем)

Документ описывает единый формат, таксономию и правила доставки уведомлений, которые формируют сервисы и доставляет **@RelloNoticeBot**.

---

## 1) Назначение

- Единый **envelope** (конверт) для всех уведомлений.
- Единый набор **типов событий** и схемы их `payload`.
- Правила **адресации** (кому слать) и **локализации** (RU/EN).
- Гарантии **надёжности**: дедуп, ретраи, TTL, троттлинг.
- Политики **приватности**: маскирование адресов, ограничение данных для работников.

---

## 2) Канал и адресация

- **Канал доставки:** только **личные сообщения** в Telegram. В группы NoticeBot **не пишет**.
- **Получатели по категориям:**
  - **Пополнение/Вывод/Депозит:** **только владелец** соответствующего кошелька/аккаунта. Работники **не видят** эти уведомления.
  - **Сделки:** обе стороны (покупатель и продавец) + **работники продавца**, если сделка идёт через его рабочую группу/чат‑группу.
  - **Арбитраж:** обе стороны получают итоговое решение; владелец‑продавец и его работники могут получать ход спора; администраторы — по своим тумблерам.
- **Недоставляемость:** если пользователь не нажал **Start** у @RelloNoticeBot — ставим `undeliverable`, делаем 3 ретрая и отправляем владельцу/админу подсказку «Откройте диалог с ботом».

---

## 3) Envelope (общая схема сообщения)

```json
{
  "notice_id": "uuid",
  "type": "deal.created",
  "severity": "info|warn|error",
  "locale": "en|ru",
  "recipients": [{ "owner_id": "..." }],
  "correlation_id": "...",
  "template_version": "v1",
  "created_at": "2025-09-21T11:22:33Z",
  "payload": { /* типоспецифичные поля */ },
  "actions": [ { "title": "Open deal", "deeplink": "https://t.me/RelloDealBot?start=deal_<id>" } ],
  "ttl_seconds": 86400
}
```

**Ограничения:** `payload` ≤ 4KB; длинные данные — ссылками (`tx_url`, `file_url`).

---

## 4) Локализация
- Поддерживаем **EN** и **RU**.
- Источники языка:
  1) `user.locale` — язык пользователя. Выставляется при регистрации по профилю Telegram и может быть **изменён пользователем в @RelloClubBot**; это влияет на язык остальных ботов по умолчанию для этого пользователя.
  2) `company.default_locale` — язык компании (владельца/продавца). Настраивается владельцем в **@RelloClubBot** и применяется ко всем ботам для сущностей компании, если нет более точной настройки.
  3) `group.group_locale` — язык конкретной чат-группы с **@RelloGroupBot**; по умолчанию наследуется от владельца компании и может быть изменён командой в группе.
- Правила выбора языка при доставке:
  - Если событие относится к **группе** (есть `group_id`) → используем `group.group_locale`.
  - Иначе, если получатель — владелец/работник компании → используем `company.default_locale`.
  - Иначе → используем `user.locale` получателя.
  - Если ничего не задано → **EN** по умолчанию.
- Envelope: поле `locale` **опционально**. Если оно не передано, @RelloNoticeBot вычисляет язык по правилам выше для **каждого получателя**. Если `locale` передан — используется для всех получателей данного уведомления.
- Шаблоны — JSON с ключами `notice.type.key`, в каждом сообщении указываем `template_version`.
- Плейсхолдеры: `{{amount_rc}}`, `{{token}}`, `{{short_addr}}`, `{{deal_id}}`; форматы — RC/крипто 6 знаков, фиат 2 знака.

---

## 5) Надёжность и лимиты

- **Дедуп:** по `notice_id` (идемпотентность).
- **Ретраи:** 3 попытки — 5с / 30с / 180с. Затем `failed`.
- **TTL:** истёк — не доставляем (актуально для «Подтвердите получение» и пр.).
- **Троттлинг:** не более 20 уведомлений в минуту на пользователя; при превышении — очередь, приоритизация по `severity`.

---

## 6) Приватность и безопасность

- Маскирование адресов: `Txxx…abcd` (первые 4/последние 4).
- Ссылки на файлы — только URL/`tg_file_id` с ограниченным TTL (файлы в системе хранятся 30 дней).
- Payload для **работников** не содержит личных контактов покупателя, только то, что требуется для обработки сделки.

---

## 7) Таксономия событий и схемы payload

Ниже — минимально необходимый набор типов и обязательные поля `payload`.

### A) Пополнение RC (`topup.*`)

- `topup.created` — создана заявка
  - `topup_id`, `owner_id`, `token`, `network`, `deposit_address`, `expires_at`
- `topup.funded` — входящий перевод обнаружен
  - `topup_id`, `tx_hash`, `tx_url`, `amount_token`, `token`
- `topup.aml.low` — AML пройден (низкий риск)
  - `topup_id`, `address_masked`, `risk` = `low`
- `topup.aml.high` — высокий риск
  - `topup_id`, `address_masked`, `risk` = `high`, `reason_codes` (array), `summary` (string), `report_ref` (string)
- `topup.credited` — зачтены RC при свипе на основной кошелёк
  - `topup_id`, `amount_rc`, `rate_pair` = `EURUSDT` (если применялось), `rate`, `tx_hash`, `tx_url`
- `topup.refund.sending` — возврат средств отправителю (минус комиссия сервиса за вывод RC)
  - `topup_id`, `amount_token_refund`, `fee_percent`, `tx_hash` (если уже есть)
- `topup.refund.failed` — авто‑возврат не удался, требуется ручная обработка
  - `topup_id`, `last_error`

### B) Вывод RC (`withdraw.*`)

- `withdraw.created`
  - `withdrawal_id`, `owner_id`, `amount_rc`, `prefer_token`, `destination_address_masked`
- `withdraw.aml.ok`
  - `withdrawal_id`, `destination_address_masked`, `risk` = `low|medium|unknown`
- `withdraw.aml.high` — отказ по high‑risk адресу назначения (с **деталями AML**)
  - `withdrawal_id`, `destination_address_masked`, `risk` = `high`, `reason_codes`, `summary`, `report_ref`
- `withdraw.sending`
  - `withdrawal_id`, `effective_token`, `amount_token`, `tx_hash` (может отсутствовать), `note` (например, замена токена)
- `withdraw.sent`
  - `withdrawal_id`, `effective_token`, `amount_token`, `tx_hash`, `tx_url`
- `withdraw.failed`
  - `withdrawal_id`, `error_code`, `error_message`

### C) Сделка (`deal.*`)

- `deal.created`
  - `deal_id`, `type` = `B2C|C2B`, `amount_rc`, `hold_side`, `seller_owner_id`, `buyer_owner_id`
- `deal.held`
  - `deal_id`, `amount_rc`, `hold_side`
- `deal.payer.sent_fiat`
  - `deal_id`, `payer_owner_id`, `evidence_file` (опц.)
- `deal.awaiting_confirmation` — снят таймер на авто‑отмену
  - `deal_id`
- `deal.success`
  - `deal_id`, `amount_rc`, `total_client_fee_rc`, `total_service_fee_rc`
- `deal.cancelled`
  - `deal_id`, `reason` = `timeout|manual`
- `deal.arbitration.opened`
  - `deal_id`, `by_owner_id`, `comment`
- `deal.arbitration.decided`
  - `deal_id`, `outcome` = `win|split|refund`, `admin_owner_id`, `comment`

### D) Депозит (`deposit.*`) — **всегда вручную**

- `deposit.requested`
  - `request_id`, `owner_id`, `kind` = `topup|withdraw`, `amount_token`
- `deposit.approved` / `deposit.rejected`
  - `request_id`, `moderator_owner_id`, `comment` (опц.)
- `deposit.executed`
  - `request_id`, `amount_token`, `tx_ref` (опц.)

### E) Безопасность (`security.*`)

- `security.lock.created`
  - `user_owner_id`, `type` = `post_restore_24h`, `until`
- `security.lock.released`
  - `user_owner_id`, `by_admin_owner_id` (если снято досрочно)
- `ban.created` / `ban.released`
  - `user_owner_id`, `by_admin_owner_id`, `reason` (для `ban.created`)

### F) Группа (`group.*`)

- `group.token.issued`
  - `company_owner_id`, `token_id`, `expires_at`
- `group.token.expired`
  - `company_owner_id`, `token_id`
- `group.linked`
  - `group_owner_id`, `company_owner_id`, `ext_telegram_group_id`

---

## 8) Кнопки/действия (CTA)

- Поддерживаются **инлайн‑кнопки** с диплинками:
  - DealBot: `https://t.me/RelloDealBot?start=deal_<id>`
  - ClubBot: `https://t.me/RelloClubBot?start=company_<id>`
- Admin/Office‑ссылки не отправляются конечным пользователям.
- Локализация `title` — по `locale` сообщения.

---

## 9) Примеры сообщений (RU/EN)

### 9.1 `topup.credited` (EN)

```
Title: Top‑up credited
Body: We received your top‑up and credited {{amount_rc}} RC. Tx: {{tx_url}}.
CTA: Open wallet → https://t.me/RelloClubBot?start=company_{{company_id}}
```

### 9.2 `withdraw.aml.high` (RU, с деталями AML)

```
Заголовок: Вывод отклонён (AML)
Текст: Адрес назначения {{destination_address_masked}} помечен как высокий риск.
Причины: {{summary}} ({{reason_codes}})
Повторите вывод на другой адрес.
```

### 9.3 `deal.awaiting_confirmation` (EN)

```
Title: Waiting for confirmation
Body: The payer marked fiat as sent. Please confirm receipt for deal {{deal_id}}.
CTA: Open deal → https://t.me/RelloDealBot?start=deal_{{deal_id}}
```

### 9.4 `deal.arbitration.decided` (RU)

```
Заголовок: Арбитраж завершён
Текст: По сделке {{deal_id}} вынесено решение: {{outcome}}. Комментарий: {{comment}}.
```

### 9.5 `security.lock.created` (EN)

```
Title: Security lock enabled (24h)
Body: After account recovery, your operations are locked until {{until}}. Contact support if this wasn’t you.
```

---

## 10) Связь с каталогом ошибок

- Для событий с ошибками (`*.failed`, `*.aml.high`, `ban.*`) включать `error_code` из \`\` и человекочитаемый `user_message`.

---

## 11) SLA и мониторинг

- Доля `failed` доставок Notice ≤ 1% (скользящее окно 1 час).
- Алерты админам: `withdraw.failed`, `topup.refund.failed`, `provider.unavailable`, `queue.backlog` (где backlog > N за M минут).

