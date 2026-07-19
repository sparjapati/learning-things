# learning-things

Personal notes on programming concepts, built up one question at a time. Notes are grouped into category folders; each category folder has its own `images/` subfolder for diagrams.

## General

- [general/qr-codes.md](general/qr-codes.md) — how a QR code encodes data as a grid, its finder/alignment/timing patterns, and Reed-Solomon error correction.
- [general/zip-file-format.md](general/zip-file-format.md) — ZIP compression (LZ77 + Huffman/DEFLATE) and the container format's per-file entries plus central directory.

## Java

- [java/hashmap.md](java/hashmap.md) — how a HashMap achieves near-O(1) lookup via bucket arrays and a hash function, how collisions are chained (and tree-ified in Java 8+), resizing/load factor, and the hashCode/equals contract.

## Kotlin

- [kotlin/kotlin-contracts.md](kotlin/kotlin-contracts.md) — how Kotlin contracts (`returns(...) implies`, `callsInPlace`) let functions like `require()`, `isNullOrEmpty()`, and `run`/`let`/`apply` enable smart casts and definite-assignment checks the compiler couldn't otherwise infer, and why an incorrect contract is unsound (declared, not verified).

## Spring Boot

- [spring-boot/spring-request-lifecycle.md](spring-boot/spring-request-lifecycle.md) — how an HTTP request flows through Tomcat, filters, DispatcherServlet, interceptors, the controller, AOP proxies, service/repository layers, and back out via message converters and ResponseBodyAdvice, plus the exception-handling path (HandlerExceptionResolver, @ExceptionHandler resolution order).

## System Design

- [system-design/async-transaction-confirmation.md](system-design/async-transaction-confirmation.md) — how a mobile app learns whether an async, event-driven transaction (e.g. order/payment) succeeded or failed: polling, long polling, WebSocket/SSE push, and mobile push notifications.
- [system-design/choosing-sql-vs-nosql.md](system-design/choosing-sql-vs-nosql.md) — decision checklist for picking SQL vs NoSQL for a given problem, with an e-commerce example.
- [system-design/idempotency.md](system-design/idempotency.md) — what idempotency means, where it shows up (HTTP methods, idempotency keys, at-least-once message delivery), the concrete mechanisms used to achieve it (dedup stores, conditional writes, unique constraints, inbox pattern), and client-generated vs server-issued keys with the "unused/consumed/unknown" three-state check needed to avoid re-processing or wrongly discarding retries.
- [system-design/message-queue-vs-pubsub.md](system-design/message-queue-vs-pubsub.md) — message queues (one consumer per message) vs pub/sub (broadcast to all subscribers).
- [system-design/monolith-vs-microservices.md](system-design/monolith-vs-microservices.md) — what "service" means (Order Service, Payment Service, etc.): independent processes/databases/deploy lifecycles talking over the network vs modules sharing one process in a monolith, plus a decision checklist for which to build.
- [system-design/mysql-table-vs-mongo-document.md](system-design/mysql-table-vs-mongo-document.md) — how a MySQL table differs from a MongoDB document, despite both being field→value stores.
- [system-design/redis-persistence-rdb-vs-aof.md](system-design/redis-persistence-rdb-vs-aof.md) — Redis persistence: RDB snapshots vs AOF write log, trade-offs and when to use each.
- [system-design/redis-single-threaded.md](system-design/redis-single-threaded.md) — how Redis works, why it's called single-threaded, why `INCR` is atomic, and use cases beyond caching.
- [system-design/saga-pattern-compensating-transactions.md](system-design/saga-pattern-compensating-transactions.md) — the Saga pattern: handling a later step (e.g. payment) failing after an earlier step (e.g. order creation) already committed, via compensating transactions, orchestration vs choreography, how to decide between them, and idempotency.
