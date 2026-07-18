# learning-things

Personal notes on programming concepts, built up one question at a time.

## Index

- [mysql-table-vs-mongo-document.md](mysql-table-vs-mongo-document.md) — how a MySQL table differs from a MongoDB document, despite both being field→value stores.
- [choosing-sql-vs-nosql.md](choosing-sql-vs-nosql.md) — decision checklist for picking SQL vs NoSQL for a given problem, with an e-commerce example.
- [redis-single-threaded.md](redis-single-threaded.md) — how Redis works, why it's called single-threaded, why `INCR` is atomic, and use cases beyond caching.
- [redis-persistence-rdb-vs-aof.md](redis-persistence-rdb-vs-aof.md) — Redis persistence: RDB snapshots vs AOF write log, trade-offs and when to use each.
- [message-queue-vs-pubsub.md](message-queue-vs-pubsub.md) — message queues (one consumer per message) vs pub/sub (broadcast to all subscribers).
- [qr-codes.md](qr-codes.md) — how a QR code encodes data as a grid, its finder/alignment/timing patterns, and Reed-Solomon error correction.
- [zip-file-format.md](zip-file-format.md) — ZIP compression (LZ77 + Huffman/DEFLATE) and the container format's per-file entries plus central directory.
- [async-transaction-confirmation.md](async-transaction-confirmation.md) — how a mobile app learns whether an async, event-driven transaction (e.g. order/payment) succeeded or failed: polling, long polling, WebSocket/SSE push, and mobile push notifications.
- [saga-pattern-compensating-transactions.md](saga-pattern-compensating-transactions.md) — the Saga pattern: handling a later step (e.g. payment) failing after an earlier step (e.g. order creation) already committed, via compensating transactions, orchestration vs choreography, how to decide between them, and idempotency.
- [monolith-vs-microservices.md](monolith-vs-microservices.md) — what "service" means (Order Service, Payment Service, etc.): independent processes/databases/deploy lifecycles talking over the network vs modules sharing one process in a monolith, plus a decision checklist for which to build.
- [idempotency.md](idempotency.md) — what idempotency means, where it shows up (HTTP methods, idempotency keys, at-least-once message delivery), the concrete mechanisms used to achieve it (dedup stores, conditional writes, unique constraints, inbox pattern), and client-generated vs server-issued keys with the "unused/consumed/unknown" three-state check needed to avoid re-processing or wrongly discarding retries.
