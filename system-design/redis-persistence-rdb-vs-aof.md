# Redis Persistence: RDB vs AOF

See also: [redis-single-threaded.md](redis-single-threaded.md) for how Redis works overall (this file expands on the "persistence" point mentioned there).

## Why persistence exists at all

Redis keeps data in RAM for speed, but RAM is volatile — a crash or restart wipes everything unless Redis has saved a copy to disk. RDB and AOF are the two mechanisms for that, and they solve it very differently.

## RDB (Redis Database file) — periodic snapshots

RDB takes a full snapshot of the entire dataset at intervals (e.g., "if at least 100 keys changed in the last 60 seconds, save"), writing it to a single compact binary file (`dump.rdb`).

- **How**: Redis forks a child process; the child writes the current in-memory dataset to disk while the parent keeps serving requests normally (uses copy-on-write memory, so it's cheap).
- **Pro**: Very fast to restart from (just load one file), and the file is compact — good for backups.
- **Con**: You can lose data since the last snapshot. Example: snapshot every 60s, Redis crashes 45 seconds after the last snapshot → those 45 seconds of writes are lost.

## AOF (Append Only File) — write log

AOF logs every write command (e.g., `SET foo bar`, `INCR counter`) to a file, in order, as it happens.

- **How**: on restart, Redis replays every command in the log from the start to rebuild the dataset exactly.
- **Pro**: Much less data loss — `fsync` can be configured to happen every second (`appendfsync everysec`, lose at most ~1s of writes) or on every single write (`appendfsync always`, effectively zero loss but slower).
- **Con**: The log file grows continuously (every write ever made), and replaying a huge log on restart is slower than loading an RDB snapshot. Redis mitigates file growth with **AOF rewriting** — periodically compacting the log to the minimal set of commands needed to reproduce the current dataset (e.g., collapsing 1000 `INCR counter` calls into one `SET counter 1000`).

## Example comparing durability

```
t=0s   SET a 1     (both RDB & AOF eventually capture this)
t=10s  SET b 2
t=20s  RDB snapshot taken here
t=25s  SET c 3
t=30s  CRASH
```

- **RDB only**: restart restores state as of t=20s — `c` is lost.
- **AOF (everysec)**: restart replays the log up to ~t=29s — `c` survives, at most the last second is at risk.

## Which to use

| | RDB | AOF |
|---|---|---|
| Data loss window | Minutes (interval-based) | ~1 second (or none, with `always`) |
| Restart speed | Fast (single snapshot) | Slower (replay log), mitigated by rewriting |
| File size | Compact | Larger (mitigated by rewrite) |
| Best for | Backups, disaster recovery, non-critical caches | Systems where data loss must be minimal |

## In practice

Redis lets you enable both simultaneously — RDB for fast backups/restarts, AOF for durability. Since Redis 4, there's also a hybrid mode where AOF rewrite uses RDB's compact binary format as its base plus a small tail of recent commands, combining AOF's durability with RDB-like file size and restart speed.
