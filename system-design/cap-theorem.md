# CAP Theorem

CAP theorem says a distributed data system can only guarantee **two out of three** properties at the same time, whenever a network partition occurs:

- **C — Consistency**: every read gets the most recent write (or an error). All nodes see the same data at the same time.
- **A — Availability**: every request gets a (non-error) response, even if it's not the latest data.
- **P — Partition tolerance**: the system keeps working even when network failures split nodes into groups that can't talk to each other.

The catch: **partitions will happen** (cables get cut, switches fail, cross-region links drop). So P isn't really optional in a real distributed system — the actual choice you're making is **C vs A when a partition occurs**.

## What "a network partition occurs" means

A **network partition** is when nodes in a distributed system can't communicate with each other due to a network failure, even though each node is individually still up and running. Nothing crashed — the nodes are alive, just cut off from each other.

This is different from a node failure: a partition specifically means the machines are healthy but isolated, not that one of them went down.

**Real-life example**: a company runs a database cluster with one node in a US data center and a replica node in a Europe data center, kept in sync over the internet link between them. One day the undersea fiber link between the US and Europe goes down (or gets congested enough to time out) — both data centers are still fully powered on and serving traffic locally, but they can no longer talk to each other. That's a network partition: two healthy groups of nodes, split apart by a broken link between them, each left to decide on its own whether to keep answering requests with what it has, or to stop and wait until the link is restored.

Other common causes: a switch or router failing between racks in the same data center, a firewall/routing misconfiguration isolating one server, or a cross-region VPN peering connection dropping.

## Real-life analogy

Imagine two bank branches (Node A and Node B) that sync your balance with each other over a phone line. You walk into Branch A and withdraw $100.

- Normally, Branch A calls Branch B, updates the balance everywhere, then confirms your withdrawal.
- Now the phone line goes down (a **partition**). You ask Branch A for your balance.
  - **Choose Consistency**: Branch A refuses to tell you anything ("can't reach Branch B, so I can't guarantee this is correct") until the line is back. Correct, but unavailable.
  - **Choose Availability**: Branch A tells you its last known balance anyway. You get an answer immediately, but it might be stale — Branch B could have processed a withdrawal it doesn't know about yet.

You can't have both "always answer" and "always correct" while the phone line is down. That's the whole theorem in one scenario.

## CP vs AP in practice

| Choice | Behavior during a partition | Example systems |
|---|---|---|
| **CP** (Consistency + Partition tolerance) | Rejects/blocks requests on the minority side rather than risk stale data | ZooKeeper, etcd, HBase, MongoDB (default config) |
| **AP** (Availability + Partition tolerance) | Keeps answering on both sides, reconciles/merges data later (eventual consistency) | Cassandra, DynamoDB, Riak |
| **CA** | Only possible without partitions — i.e. a single-node or non-distributed system. Not a real option once you're distributed. | Traditional single-node RDBMS |

## How to decide: CP or AP for your system?

| Question | Lean CP | Real-life example (CP) | Lean AP | Real-life example (AP) |
|---|---|---|---|---|
| Is stale/conflicting data actually unsafe (money, inventory, locks/leader election)? | Yes | A bank rejects a withdrawal it can't confirm against the latest balance, rather than risk overdrawing the account. | No — a slightly outdated read is tolerable | An e-commerce product page shows a "4.8★, 1,203 reviews" count that's a few minutes stale — nobody is harmed by that. |
| Can the operation be corrected later if it turns out to be wrong (e.g. reconciled, refunded, merged)? | No — must be right the first time | A flight seat assignment can't be double-booked and then "fixed later" — the seat is a single physical resource. | Yes | Two edits to the same shopping cart from two devices can be merged (union of items) once both sides reconnect — no harm done by the temporary conflict. |
| Is it acceptable for the system to refuse requests during a network issue rather than answer with possibly-stale data? | Yes | A stock trading system halts order placement during an exchange outage rather than accept trades against a price it can't verify. | No — uptime matters more than perfect freshness | A social media app keeps showing your feed (even a slightly stale one) during a partial outage rather than showing an error page. |
| Do you need coordination guarantees (e.g. only one leader, no double-booking the same seat)? | Yes | A movie ticketing system uses a lock so only one buyer can grab the last seat in a showing. | No | A "likes" counter on a post doesn't need a lock — two regions incrementing it concurrently and reconciling later is fine. |
| Is the workload read-heavy with tolerance for eventual convergence (e.g. social feed likes, product view counts)? | No | N/A — this scenario is specifically what AP is good at. | Yes | A video platform's "view count" is allowed to be approximate and catch up over time; blocking video playback to keep it perfectly accurate would be a bad trade. |

**Worked examples**

- **Bank ledger / payments** → CP. Two branches must never show different balances after a confirmed transfer — better to reject a request during a partition than let both sides process conflicting withdrawals.
- **Distributed lock / leader election (ZooKeeper, etcd)** → CP. Two nodes both believing they're "the leader" during a partition causes split-brain, which is worse than one side being briefly unavailable.
- **Shopping cart / product catalog (DynamoDB, Cassandra)** → AP. If a cart shows an item that's one second stale, that's fine — the store staying up and responsive matters more than instant consistency, and conflicts get merged on reconnect.
- **Social media like counts / view counts** → AP. Nobody notices if a like count is briefly a few seconds behind across regions; refusing to serve the page during a partition would be a worse user experience than a stale number.

## Why it matters

This is the theoretical backbone behind the tradeoffs discussed in [choosing-sql-vs-nosql.md](choosing-sql-vs-nosql.md) — a distributed database's claim of "always consistent and always available" only holds as long as nothing ever partitions, which isn't realistic at scale.