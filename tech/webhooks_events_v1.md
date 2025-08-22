# События/вебхуки (v1) — кратко
События: `deal.created`, `deal.updated`, `pay_in.detected`, `payout.sent`, `payout.confirmed`, `trust.session.closed`, `eod.settlement.sent/confirmed`, `dispute.opened/decided`.
Подпись: HMAC‑SHA256, timestamp, идемпотентность. Переотправка с экспоненциальной паузой. Реплей: `GET /events` + `POST /webhooks/{id}/replay`.
