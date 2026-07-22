# Message Queue vs Pub/Sub vs Event Bus

See also: [redis-single-threaded.md](redis-single-threaded.md), which lists both "message/task queues" (`LPUSH`/`BRPOP`) and "pub/sub messaging" (`PUBLISH`/`SUBSCRIBE`) as separate Redis use cases — this file explains the general distinction between the two patterns.

## The core confusion

Both let one part of a system notify another without a direct function call. The difference is what happens to the message after it's sent, and whether the receiver has to be listening at that exact moment.

## Message Queue: messages are stored and consumed once

A message queue holds messages until someone picks them up. Once a consumer takes a message off the queue, it's gone (removed) — no other consumer gets that same message.

```
Producer → [Queue: msg1, msg2, msg3] → Consumer pulls msg1 (queue now: msg2, msg3)
```

- **Delivery**: guaranteed, even if no consumer is running right now — the message just waits in the queue.
- **Consumption**: typically one consumer per message (if 3 workers read the same queue, each message goes to only one of them — this is how work gets distributed across workers).
- **Use case**: task distribution — e.g. "process this image," "send this email." The job should be done exactly once, by whichever worker is free.
- **Example**: an e-commerce site pushes "new order #123" onto a queue; any one of 5 worker processes picks it up and processes the order exactly once.

## Pub/Sub: messages are broadcast, not stored

A publisher sends a message to a channel; every subscriber currently listening to that channel gets a copy of it, simultaneously.

```
Publisher → channel "news" → Subscriber A gets it
                            → Subscriber B gets it
                            → Subscriber C gets it
```

- **Delivery**: only to subscribers listening right now. If nobody is subscribed when the message is published, it's lost (in basic pub/sub — some systems like Kafka add persistence to change this).
- **Consumption**: every subscriber gets every message — broadcast, not divided up.
- **Use case**: notifying multiple independent parts of a system about an event — e.g. "user logged in" should trigger an analytics logger AND a live notification AND a cache invalidation, all independently, all getting the same event.
- **Example**: a stock price update is published to channel `AAPL`; every dashboard currently displaying AAPL's price receives it live. A dashboard opening 5 seconds later missed that tick (unless a mechanism replays history).

## Side-by-side

| | Message Queue | Pub/Sub |
|---|---|---|
| Who gets each message | Exactly one consumer | Every current subscriber |
| If no one's listening | Message waits until someone consumes it | Message is lost (in basic pub/sub) |
| Purpose | Distribute work / tasks | Broadcast events/notifications |
| Analogy | A ticket counter queue — one ticket, one person served | A radio broadcast — anyone tuned in hears it, tune in late and you missed it |

## Where does an "Event Bus" fit? Is it related to these?

Yes — but it's not a third delivery semantic alongside queue and pub/sub. "Event bus" is a name for a *component* that routes events from publishers to subscribers, and it's almost always built on **pub/sub semantics** (broadcast to whoever's currently listening), not queue semantics. What actually varies between event bus implementations is **scope** and **routing**:

- **In-process event bus**: lives inside a single application, decoupling classes from each other with no network hop at all. Spring's `ApplicationEventPublisher` + `@EventListener` (see [spring-execution-contexts-and-hooks.md](../spring-boot/spring-execution-contexts-and-hooks.md)) and Guava's `EventBus` are both this — a publisher calls one method, any number of listener methods elsewhere in the same JVM react, and there's zero durability if nothing's listening.
- **Distributed event bus**: a managed service playing the same broadcaster role but across services/processes — AWS EventBridge, Azure Event Grid, Google Eventarc. These add **rule-based routing**: instead of every subscriber getting everything on a channel, you write rules matching on event content (e.g. "route events where `detail-type = OrderPlaced`") to specific targets. That's the one real feature distinguishing "event bus" from plain pub/sub — a flat pub/sub channel (Redis `PUBLISH`/`SUBSCRIBE`) has no content-based routing; every subscriber to a channel gets everything published there.

```
Event Bus (distributed):
Publisher → bus → rule: detail-type=OrderPlaced   → Lambda A
                → rule: detail-type=PaymentFailed  → SQS queue
                → rule: catch-all                  → Logging service
```

So: **message queue** and **pub/sub** are delivery *semantics*. **Event bus** is a *component category* that almost always implements pub/sub semantics (sometimes layering queue-like persistence and content-based routing on top), at either in-process or cross-service scope.

| | Event Bus |
|---|---|
| Who gets each message | Every subscriber whose rule matches (a filtered subset of "every subscriber") |
| If no one's listening | Depends on implementation — durable/retried if backed by a managed broker (EventBridge), silently lost if in-process (no listener registered, nothing happens) |
| Purpose | Decouple publishers from reactors, typically with content-based routing to different targets |
| Analogy | A company mailroom that reads the address on each letter and routes it to the right department, rather than reading every letter aloud over the PA system |

## Spring Boot in practice: `ApplicationEventPublisher` vs JMS

- **`ApplicationEventPublisher` + `@EventListener`** — Spring's in-process event bus (introduced above). Pure pub/sub semantics: `publisher.publishEvent(event)` hands the event to every `@EventListener` method whose parameter type matches, synchronously, on the same thread, in the same transaction, by default. There's no queue anywhere — with zero listeners, or mid-restart, the event is just gone. No broker, no serialization, no network hop, since everything stays inside one JVM. The `@Async`/`@TransactionalEventListener` variants of this mechanism are covered in [spring-execution-contexts-and-hooks.md](../spring-boot/spring-execution-contexts-and-hooks.md).
- **JMS** (Java Message Service, via `spring-boot-starter-artemis`/`spring-boot-starter-activemq`) — Spring's abstraction over an actual **message broker process** (ActiveMQ, Artemis), separate from your application's JVM entirely. `JmsTemplate.convertAndSend(destination, message)` sends; `@JmsListener(destination = "...")` receives. JMS supports both semantics natively, as two destination types:
  - `Queue` destination — classic message-queue semantics: one message, delivered to exactly one consumer, even with several `@JmsListener`s competing on that queue.
  - `Topic` destination — pub/sub semantics: every consumer currently subscribed to that topic gets its own copy.

```java
// ApplicationEventPublisher — in-process, pub/sub only, no broker
@Service
class OrderService {
    private final ApplicationEventPublisher publisher;
    void placeOrder(Order order) {
        publisher.publishEvent(new OrderPlacedEvent(order.getId())); // same JVM, same thread by default
    }
}

// JMS — out-of-process, broker-backed, queue OR topic depending on destination
@Service
class OrderJmsService {
    private final JmsTemplate jmsTemplate;
    void placeOrder(Order order) {
        jmsTemplate.convertAndSend("order.queue", order.getId()); // durable, survives a restart
    }
}

@JmsListener(destination = "order.queue")
void onOrder(Long orderId) { ... } // one of N competing listeners gets each message
```

| | `ApplicationEventPublisher` | JMS |
|---|---|---|
| Semantics | Pub/sub only | Queue *or* topic — chosen per destination |
| Process boundary | In-process (same JVM) | Out-of-process (a separate broker: ActiveMQ/Artemis) |
| Durability if nobody's listening | None — the event is just gone | Durable — the broker holds it until a consumer (or a reconnecting durable topic subscriber) picks it up |
| Typical use | Decoupling classes inside one app (e.g. "send confirmation email after order commits") | Cross-service/cross-instance work distribution or notification that must survive a restart |

## Why the line can blur

Some systems (Kafka, Redis Streams, and — as above — JMS itself via its queue/topic destination types) combine both ideas — messages are persisted like a queue, and multiple independent consumer groups can each get their own full copy of the stream (broadcast-like), while within a consumer group messages are still distributed one-per-consumer (queue-like). So "queue vs pub/sub" is really about two separate properties — persistence, and one-vs-all delivery — and modern systems mix and match them rather than being purely one or the other.

See also [async-transaction-confirmation.md](async-transaction-confirmation.md) for how a client finds out the result once patterns like these are used to process something asynchronously.
