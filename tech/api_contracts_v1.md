# API‑контракты (v1) — кратко
- `POST /offers` / `GET /offers` / `POST /offers/{id}/accept`.
- `GET /deals/{id}` / `POST /deals/{id}/cancel`.
- `GET /treasury/payouts` / `POST /treasury/payouts/{id}/approve` / `{id}/send`.
- `GET /trust/pairs` / `POST /trust/pairs` / `POST /trust/pairs/{id}/pause`.
- Аутентификация: Short‑Token (deeplink) + tg_id; идемпотентность через заголовок `Idempotency-Key`.
(Полные схемы JSON — добавим при имплементации контроллеров.)
