# Distributed Rate Limiter Architecture (Uber's Control Plane Approach)

## Overview

Traditional distributed rate limiters often rely on a centralized datastore (such as Redis) to make a decision for every incoming request. While simple to implement, this approach introduces latency, infrastructure complexity, and scalability challenges at very high request volumes.

Uber designed a different architecture where **rate limiting decisions are made locally**, while **global rate limiting policies are computed asynchronously** by a centralized control plane.

This document explains this architecture and the design decisions behind it.

---

# Goals

The rate limiter should:

- Protect downstream services from overload.
- Ensure fair resource sharing between clients.
- Prevent cascading failures.
- Scale to millions of requests per second.
- Minimize latency added to every request.
- Avoid centralized bottlenecks.

---

# Common Rate Limiting Strategies

Before looking at Uber's specific architecture, it helps to know the standard algorithms most rate limiters are built on. Each makes a different trade-off between accuracy, memory cost, and burst tolerance.

## 1. Fixed Window Counter

Time is divided into fixed-size windows (e.g. 1 minute). A single counter tracks requests in the current window and resets when the window rolls over.

```
Window: 00:00 - 01:00   Counter: 87 / 100
Window: 01:00 - 02:00   Counter: 0 / 100 (reset)
```

**Pros**: trivial to implement, one counter per key.

**Cons**: allows a burst at window boundaries. A client can send 100 requests in the last second of one window and another 100 in the first second of the next — 200 requests in ~2 seconds, even though the configured limit is 100/minute.

---

## 2. Sliding Window Log

Every request's timestamp is logged. On each new request, timestamps older than `now - window size` are discarded, then the remaining count is compared to the limit.

```
Request timestamps in the last 60s: [12:00:01, 12:00:15, 12:00:40, ...]

Discard anything older than (now - 60s), count what's left.
```

**Pros**: fully accurate — no window-boundary burst problem.

**Cons**: memory cost grows with request volume, since every request's timestamp has to be stored for the duration of the window.

---

## 3. Sliding Window Counter

A hybrid: keep fixed-window counters, but weight the previous window's count by how much it overlaps the current sliding view, instead of storing every timestamp.

```
Previous window count: 80, current window count: 20

30% of the previous window still overlaps the sliding view

Effective count = 20 + (0.3 * 80) = 44
```

**Pros**: close approximation of the sliding log's accuracy, at fixed-window-like memory cost.

**Cons**: an approximation, not exact — can drift slightly in edge cases.

---

## 4. Token Bucket

A bucket holds up to N tokens (the burst capacity) and refills at a fixed rate. Each request consumes one token; if the bucket is empty, the request is rejected (or delayed).

```
Bucket capacity: 10 tokens
Refill rate: 1 token/sec

Burst of 10 requests: allowed instantly (bucket had 10 tokens)
11th request: rejected until a token refills
```

**Pros**: allows controlled bursts up to the bucket size while capping the long-term average rate. Widely used (Stripe, AWS API Gateway).

**Cons**: in a distributed setting, every node needs the same up-to-date token count and refill rate — exactly the synchronization problem Uber's design in this document is built to avoid (see "Why Not Token Bucket?" below).

---

## 5. Leaky Bucket

Conceptually similar to token bucket, but framed as a queue: requests enter a bucket that "leaks" (processes) at a constant fixed rate. If the bucket is full, new requests are dropped.

```
Incoming: bursty          Outgoing: constant rate

   |||| ----->   Bucket   ----->   | | | | | | |
```

**Pros**: guarantees a smooth, constant output rate — useful when the downstream system can't tolerate any burst at all.

**Cons**: unlike token bucket, it gives no burst allowance by design; implementing it as an actual queue (rather than just dropping) adds latency to accepted requests.

---

**Where Uber's approach differs from all of these**: the five strategies above all answer "how much quota does *this one client* have left?" — they're per-key algorithms (per user, per IP, per API key). Uber's control-plane design below doesn't track per-client quota with any of these algorithms at all — it computes one global "drop X% of everything" policy from aggregate traffic vs. capacity, and applies it probabilistically, trading exact per-client fairness for near-zero synchronization cost at massive scale.

---

# Problems with Traditional Redis-based Rate Limiting

Typical architecture:

```
Client
   |
   v
Redis
   |
Decision
   |
   v
Service
```

For every request:

1. Increment a counter in Redis.
2. Check whether the limit is exceeded.
3. Return Allow/Reject.

## Drawbacks

### 1. Additional Network Hop

Every request requires communication with Redis before reaching the application.

```
Client
    |
 Redis
    |
 Service
```

instead of

```
Client
    |
 Service
```

This increases:

- Request latency
- Network traffic
- CPU usage

---

### 2. Redis Becomes the Bottleneck

If the system handles

```
500 Million Requests/sec
```

then Redis must process

```
500 Million Counter Updates/sec
```

The rate limiter itself becomes the scalability bottleneck.

---

### 3. Operational Complexity

Operating Redis clusters requires:

- Sharding
- Replication
- Failover
- Monitoring
- Capacity planning
- Scaling

Instead of solving application problems, engineers spend time maintaining Redis infrastructure.

---

# High-Level Architecture

Uber separates the system into:

- **Data Plane**
- **Control Plane**

## Data Plane

Responsible for serving requests.

Requirements:

- Extremely fast
- No remote dependency
- No Redis
- No centralized coordination

## Control Plane

Responsible for:

- Collecting traffic metrics
- Computing rate limits
- Distributing updated policies

---

# Architecture

```
                 Controller
            (Global Rate Limits)
                     ▲
                     │
          Aggregators (Per Zone)
                     ▲
                     │
      Service Mesh Clients / Proxies
                     ▲
                     │
                  Applications
```

Only aggregated metrics travel upward.

Rate limit policies travel downward.

Individual requests never leave the local machine.

---

# Components

## 1. Service Mesh Client (Proxy)

Every request passes through the service mesh.

Responsibilities:

- Count requests.
- Apply current rate limiting policy.
- Report traffic statistics periodically.

Example:

```
Pricing Service

↓

Maps Service

100 Requests/sec
```

The proxy reports:

```
100 Requests/sec
```

to the zone aggregator.

No per-request communication is required.

---

## 2. Zone Aggregator

Each zone aggregates metrics from all hosts.

Example:

```
Host A : 120 RPS
Host B : 180 RPS
Host C : 100 RPS

Total

400 RPS
```

The aggregated value is forwarded to the controller.

---

## 3. Global Controller

The controller receives traffic from every zone.

Example:

```
Zone A : 400 RPS
Zone B : 700 RPS
Zone C : 300 RPS
```

Total traffic:

```
1400 RPS
```

Based on configured limits, it computes the required throttling policy.

---

# Why Local Decisions?

Each proxy only observes its own traffic.

Example:

```
100 Proxies

Each receives

10 Requests/sec
```

Global traffic is

```
1000 Requests/sec
```

No single proxy knows the global request rate.

Therefore:

- Metrics are aggregated globally.
- Decisions are distributed back to proxies.

---

# Rate Limiting Policy

Instead of sending:

```
Allow only 100 requests/sec
```

the controller sends:

```
Drop 15% of requests
```

Every proxy independently applies this policy.

---

# Probabilistic Dropping

Suppose:

Capacity

```
1000 RPS
```

Incoming traffic

```
1200 RPS
```

Excess traffic

```
200 RPS
```

Required drop ratio:

```
200 / 1200 = 16.7%
```

Controller sends:

```
Drop Ratio = 16.7%
```

Every proxy randomly rejects approximately 16.7% of requests.

Example:

```
Incoming

100 Requests

↓

Reject 17

↓

Allow 83
```

Across thousands of proxies, the average converges very closely to the desired global rate.

---

# Why Probabilistic Dropping?

Advantages:

- No synchronization between proxies.
- No distributed counters.
- No global locks.
- Extremely scalable.
- Constant-time decision for every request.

Each request is processed independently.

---

# Why Not Token Bucket?

A distributed token bucket requires every proxy to maintain the correct token refill rate.

Whenever capacity changes:

- Token refill rates must be synchronized.
- Configuration must be pushed continuously.
- Distributed coordination becomes more complicated.

Uber simplified the design by using:

```
Single Policy

↓

Drop Percentage
```

instead of synchronizing token buckets.

---

# Request Flow

```
Incoming Request

        |

Proxy receives request

        |

Read current drop percentage

        |

Generate random number

        |

+---------------------------+
| Random < Drop Percentage? |
+---------------------------+

       | Yes                     | No
       |                         |
Reject Request             Forward Request
```

Decision time is O(1).

No network communication is required.

---

# Metric Collection Flow

```
Proxy

↓

Traffic Statistics

↓

Zone Aggregator

↓

Global Controller

↓

Updated Drop Ratio

↓

Zone Aggregator

↓

Proxy
```

Only metrics and policies are exchanged.

Requests never participate in this communication.

---

# Handling Traffic Spikes

Metrics are collected periodically (approximately every second).

Example:

```
Traffic Spike

↓

Proxy

↓

Aggregator

↓

Controller

↓

Updated Policy

↓

Proxies
```

The response to sudden traffic changes is not instantaneous.

A small amount of excess traffic may pass before the updated policy is propagated.

This trade-off significantly reduces system complexity while maintaining excellent scalability.

---

# Failure Handling

If the controller becomes unavailable:

- Existing policies continue to be used.
- No new policies are distributed.
- The system fails open instead of blocking all traffic.

Failing open prevents the rate limiter from becoming a single point of failure.

---

# Automatic Configuration

Initially, rate limits were configured manually.

Example:

```yaml
caller: pricing-service
target: maps-service
limit: 5000 RPS
```

As systems evolve, manual configuration becomes difficult to maintain.

Uber introduced an automated configurator that:

1. Collects historical traffic.
2. Analyzes usage patterns.
3. Computes safe rate limits.
4. Generates updated configurations.
5. Pushes them to controllers.

This minimizes manual tuning.

---

# Advantages

- No Redis dependency in the request path.
- Extremely low request latency.
- Horizontally scalable.
- No centralized request bottleneck.
- Simple local decision making.
- Global visibility through aggregated metrics.
- Automatic policy updates.
- Easy integration with service mesh proxies.

---

# Limitations

- Decisions are eventually consistent.
- Sudden spikes are handled with a small delay.
- Accuracy depends on metric reporting frequency.
- Random dropping may reject some otherwise acceptable requests.
- Requires a control plane infrastructure.

---

# Comparison

| Traditional Redis Limiter | Uber's Architecture |
|---------------------------|---------------------|
| Per-request Redis lookup | Local decision |
| High latency | Very low latency |
| Redis bottleneck | No centralized bottleneck |
| Strongly consistent counters | Eventually consistent metrics |
| Operational overhead | Lightweight data plane |
| Difficult to scale | Designed for hyperscale |

---

# Key Design Principle

Separate the **request path** from the **decision-making process**.

## Traditional Approach

```
Request

↓

Redis

↓

Decision

↓

Application
```

Every request depends on a centralized component.

---

## Uber's Approach

```
Request

↓

Local Decision

↓

Application


Background

↓

Collect Metrics

↓

Global Controller

↓

Compute Policy

↓

Distribute Policy
```

Requests remain fast because all expensive computation happens asynchronously in the control plane.

---

# Summary

Uber's distributed rate limiting system replaces centralized request coordination with a control-plane architecture.

Instead of querying Redis for every request, each proxy makes local rate limiting decisions using the latest policy distributed by the controller. Traffic statistics are periodically aggregated to compute new global policies, which are then propagated back to the proxies.

This architecture significantly reduces request latency, eliminates centralized bottlenecks, and scales efficiently to hundreds of millions of requests per second while still enforcing global rate limits.