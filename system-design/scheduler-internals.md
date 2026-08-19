# How Schedulers Actually Work (Why They Don't Poll Every Millisecond)

## The short answer

A naive design — loop forever, check the clock, ask "is it time yet?" — would burn CPU constantly for no reason, since almost every check says "not yet." That naive design (**polling**) does exist historically, but virtually every real scheduler — from the OS kernel up to an app's `@Scheduled` job — replaced it decades ago with a **deadline-based sleep** design instead.

## The actual mechanism: sleep exactly until the next deadline

**Analogy: an alarm clock, not a person staring at a clock.** You don't watch the clock every second waiting for 7:00 AM — you set an alarm for exactly 7:00 AM and go to sleep. The alarm hardware doesn't "check" anything repeatedly either; it's armed to fire once, at that instant. With several alarms, you effectively only care about whichever one is soonest — that's the whole trick.

Most in-process schedulers (Java's `ScheduledThreadPoolExecutor` — which both `java.util.Timer`'s replacement and Spring's `@Scheduled` sit on top of — plus JS event loop timers) work like this:

1. Every scheduled task carries a **next fire time** (an absolute timestamp).
2. All pending tasks sit in a **min-heap ordered by that timestamp** — the soonest-due task is always at the root, `O(log n)` to insert/remove.
3. The scheduler's worker thread does **not** loop-and-check. It peeks the earliest deadline, computes `delay = deadline - now`, and **blocks** on a wait/condition-variable for exactly that duration (`LockSupport.parkNanos(delay)` in Java, or the equivalent OS primitive).
4. The thread wakes up in exactly one of two cases: the sleep naturally elapses (task is due), or a *new* task gets scheduled with an earlier deadline than the one currently being waited on — which explicitly interrupts the sleep early.
5. It runs whatever's now due, re-peeks the heap, and sleeps again for the new soonest delay.

Net effect: the CPU does **zero work** between now and the next real deadline — no polling loop, no wasted wake-ups.

## Pseudocode shape

```
minHeap = []   # tasks ordered by nextFireTime

function schedule(task, delay):
    minHeap.insert(task, now() + delay)
    wakeSchedulerThreadIfThisIsSoonerThanCurrentSleep()

function schedulerLoop():
    while true:
        if minHeap.isEmpty():
            sleepUntilWoken()                  # blocks indefinitely, no task pending
        else:
            next = minHeap.peekEarliest()
            delay = next.fireTime - now()
            if delay <= 0:
                run(minHeap.popEarliest().task)
            else:
                sleepFor(delay)                # blocks for exactly `delay`, or wakes early on a new earlier task
```

## When there are *millions* of timers: timer wheels

A min-heap's `O(log n)` per operation is fine for thousands of tasks, but breaks down with millions of short-lived timers (e.g. a per-connection timeout on every open socket in a busy server). A **timer wheel** gives `O(1)` insert/expire instead.

**Analogy: a rotating parking-meter dial.** Picture a circular array of buckets, one per time slot (say, one bucket per millisecond, wrapping around). A task due in 500ms drops straight into the bucket 500 slots ahead of "now" — no comparisons needed. A single pointer sweeps one bucket per tick and only ever has to look at whatever's sitting in that *one* bucket it's currently passing, never the others. Netty's `HashedWheelTimer`, the Linux kernel's timer wheels, and Kafka's request-timeout "purgatory" all use this.

## The bottom layer: hardware timer interrupts

Even "sleep until deadline" ultimately needs the OS to wake a sleeping thread at the right instant without polling *itself*. Modern CPUs have a programmable hardware timer (APIC timer / TSC-deadline mode on x86) that can be armed to fire a single interrupt at an arbitrary future instant. The kernel finds the single earliest deadline across every sleeping thread in the whole system and programs the hardware for exactly that point — the **tickless kernel** design (Linux's dynamic ticks, `NO_HZ`, since ~2007). Older kernels used a fixed periodic tick (a hardware interrupt every 4–10ms, ~100–250Hz) where the OS genuinely *did* wake up on a schedule and check "did anything become due" — closer to the polling model — but that wastes power/CPU when nothing's due, which is exactly why it was replaced.

## Decision-style summary

| Approach | When it's used | Real example |
|---|---|---|
| Min-heap / delay queue | General purpose, thousands of tasks | Java `ScheduledThreadPoolExecutor`, Spring `@Scheduled`, JS event loop timers |
| Timer wheel | Millions of short-lived timers, need O(1) insert/expire | Netty's `HashedWheelTimer`, Linux kernel timers, Kafka's timeout purgatory |
| Fixed periodic tick (true polling) | Legacy/simple systems where granularity doesn't matter | Old kernel 100Hz clock tick, simple cron daemons checking once a minute |

## Tying it back to Spring

`@Scheduled`'s `TaskScheduler` doesn't evaluate a cron expression every second to ask "does now match?" It computes the **single next fire time** from the cron expression once, schedules exactly there via the delay-queue executor described above, and only recomputes the next time after that run completes.
