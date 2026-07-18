# Message Queue vs Pub/Sub Messaging

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

## Why the line can blur

Some systems (Kafka, Redis Streams) combine both ideas — messages are persisted like a queue, and multiple independent consumer groups can each get their own full copy of the stream (broadcast-like), while within a consumer group messages are still distributed one-per-consumer (queue-like). So "queue vs pub/sub" is really about two separate properties — persistence, and one-vs-all delivery — and modern systems mix and match them rather than being purely one or the other.
