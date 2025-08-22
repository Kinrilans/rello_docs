# Схема БД (v2) — обзор
Сущности: Company, Member, Wallet, Offer, Deal, TrustPair, TrustSession, EODSettlement, Deposit, DepositLedger, PayoutTask, IncomingTx, DeeplinkTicket, AuditLog.
Ключевые поля: суммы в NUMERIC(38,18); статусы — enum; индексы по company/status/created_at; уникальность trust_pair (a↔b), wallet(address,company). Диаграмма ER — см. приложенный перечень таблиц в плане MVP.
