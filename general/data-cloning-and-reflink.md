# Data Cloning for Dev/Test Environments (Reflink & Copy-on-Write)

> "Copying is cheap; copying *data* is expensive."
>
> *A reflink copy is the first without the second — a new file that points at the old file's blocks until somebody writes.*

## 1. The problem: every environment wants its own copy of production-shaped data

A team running many non-production environments — **devboxes** (one per engineer), **sandboxes** (short-lived, per-branch or per-feature), and **staging/UAT machines** (shared, pre-release) — hits the same wall: each of them needs a realistic database to be useful, and "realistic" means production-sized.

Giving each environment a full physical copy doesn't scale on either axis:

| | Full copy per environment | Consequence |
|---|---|---|
| **Time** | Restoring a few hundred GB of MySQL/Mongo data takes tens of minutes to hours | An engineer asking for "fresh data" loses half a day |
| **Space** | 20 devboxes × 300 GB = 6 TB | The volume fills up long before the team does |

**Data cloning** solves both by making the copy a *bookkeeping* operation instead of a *data* operation. One masked copy of production is prepared centrally; every environment then gets a near-instant, near-free clone of it via the file system's **reflink** support.

**Analogy.** A library keeps one master reference binder (**the shared snapshot on disk**). Instead of photocopying the whole binder for each reader, it hands each reader an index card saying "your binder *is* the master binder" (**the reflink — a new file whose extents point at the same blocks**). Reading costs nothing extra. The moment a reader scribbles on page 42, the library photocopies *only page 42* and staples it into that reader's card (**copy-on-write of just the modified blocks**); every other page is still the master's (**the untouched shared extents**). Nobody else sees the scribble, and the master is never altered.

### When does this technique apply?

- **Devboxes** — an engineer wants a database that looks like production, on demand, several times a week ("reset my box"). The clone is seconds, so resetting is cheap enough to be routine rather than a ticket.
- **Sandboxes / ephemeral environments** — a per-PR or per-branch environment that lives for hours. A full restore would dominate its entire lifetime; a clone makes the environment's startup cost effectively the container boot time.
- **Staging / UAT** — a shared pre-release environment refreshed before a release cycle or a UAT round, so testers validate against current-shaped data.
- **Destructive testing** — running a migration, a bulk backfill, or a data-fix script against a clone, then throwing it away and re-cloning. Because writes only diverge, "throw it away" costs nothing.
- **Debugging a production data bug** — clone the snapshot, reproduce locally against the same rows, without touching production.

One masked snapshot is built once, centrally; every environment then clones it in seconds:

![Data cloning pipeline: prod → masking job → shared snapshot → reflink clone → devbox/sandbox/staging](images/data-cloning-reflink-pipeline-eraser.png)

### How to decide how to seed a non-prod environment

| Question to ask about your setup | Reflink clone from a shared snapshot | Dump & restore per environment | One shared non-prod DB for everyone | Synthetic / fixture data |
|---|---|---|---|---|
| **How often is the data reset, and how long can a reset take?** | Often, and it must be minutes. *Real-life:* a devbox fleet where engineers re-clone several times a week. | Rarely — a quarterly refresh where an hours-long restore is tolerable. *Real-life:* a single UAT box refreshed once per release. | Never reset individually; it's just there. *Real-life:* a shared QA database a whole team points at. | Every test run rebuilds it in seconds. *Real-life:* CI spinning up a container and running Flyway + seed scripts. |
| **How many independent copies do you need at once?** | Many, and they must not see each other's writes. *Real-life:* 20 devboxes + 8 staging + 4 UAT off one snapshot. | A handful — the storage cost is linear, so it caps out fast. *Real-life:* two long-lived pre-prod environments. | Exactly one; isolation isn't required. *Real-life:* a read-mostly integration environment. | Unlimited — each is tiny. *Real-life:* parallel CI jobs with Testcontainers. |
| **Does the test need production-*shaped* data (volume, skew, weird legacy rows)?** | Yes — the whole point is real cardinality and real edge-case rows. *Real-life:* reproducing a slow query that only misbehaves at production row counts. | Yes, same reason — you just pay full price per copy. *Real-life:* a performance-test environment loaded once and left alone. | Yes, but shared, so one tester's write corrupts another's scenario. *Real-life:* a demo environment. | No — a handful of hand-built rows is clearer and deterministic. *Real-life:* unit/integration tests asserting exact outputs. |
| **What is the storage and file-system situation?** | Source and destination on **one** reflink-capable file system (XFS `reflink=1`, Btrfs, APFS, ReFS). *Real-life:* snapshot and live data dirs on the same XFS volume. | Anything — works on ext4, across hosts, across clouds. *Real-life:* restoring a `mysqldump` onto a laptop. | Anything — there's only one copy. *Real-life:* a managed RDS instance. | Anything. *Real-life:* an in-memory H2/SQLite in tests. |
| **Who is allowed to see the data?** | Masked/anonymised once, centrally, before the snapshot is published — every clone inherits the masking. *Real-life:* PII scrubbed in the snapshot-building job. | Masking must be re-applied per restore, which people forget. *Real-life:* an ad-hoc dump someone took "just to debug". | One masked copy, tightly access-controlled. *Real-life:* a locked-down analytics replica. | No real data at all — the safest option by construction. *Real-life:* an open-source project's demo dataset. |

Rule of thumb: **synthetic data for tests that assert behaviour; cloned production-shaped data for tests that assert the system survives reality.** Most teams need both, not one.

## 2. Reflink

A reflink copy creates a new file that shares the original file's disk blocks. No file data is read or written — only metadata.

Normal copy:
```
file1 ──► [A][B][C][D]
file2 ──► [A][B][C][D]      ← data duplicated
```

Reflink copy:
```
file1 ──┐
        ├──► [A][B][C][D]   ← one set of blocks, shared
file2 ──┘
```

- **Time**: seconds, regardless of size (metadata only).
- **Space**: near zero at copy time.
- **Command**: `cp --reflink=always` (fail if unsupported) or `cp --reflink=auto` (fall back to a full copy).

## 3. Copy-on-write (CoW)

When a shared block is written, the file system writes the new data to a fresh block and repoints only the file being written.

```
file1 ──► [A][B][C ][D]
file2 ──► [A][B][C'][D]     ← only C' is new; A, B, D remain shared
```

- The original file never changes.
- Disk usage grows only by the blocks that differ, so a devbox that runs a week of tests still costs a fraction of a full copy.
- XFS allocates CoW space in chunks (CoW extent size hint, default up to ~128 KB).

### Example: inserting a row into a `wallet` table

- InnoDB modifies one or a few 16 KB pages in `wallet.ibd` (= 4 × 4 KB XFS blocks each).
- Only those blocks are copied; the rest of `wallet.ibd` stays shared.
- The same write also touches `#innodb_redo/*`, doublewrite (`#ib_16384_*.dblwr`), `undo_00*`, and `binlog.*`; each gets new blocks the same way.

### Shared blocks, not shared files

- The cloned `wallet.ibd` does **not** reference the *path* of the snapshot's `wallet.ibd`.
- XFS keeps a reference count per block.
- Deleting or replacing the snapshot file just drops the count; the clone keeps its blocks and stays valid.

## 4. Who does what

| Layer | Role |
|---|---|
| `cp -a --reflink` | Requests the clone via `ioctl(FICLONE)` per file. |
| Linux kernel / XFS | Creates shared block references, keeps refcounts, performs CoW on write. |
| MySQL (or Mongo/Redis/ES) | Unaware. Reads and writes its data files as normal files. |

Requirements:
- The file system must support reflink (XFS needs `reflink=1` at `mkfs` time).
- Source and destination must be on the **same** file system — this is why the snapshot and the live data directory are deliberately placed on one volume.

## 5. File system support

### Linux

| File system | Reflink | Notes |
|---|---|---|
| XFS | Yes | Requires `reflink=1` (default in modern `mkfs.xfs`). |
| Btrfs | Yes | CoW by design; also subvolume snapshots. |
| bcachefs | Yes | |
| OCFS2 | Yes | |
| ZFS (OpenZFS) | Partial | Block cloning since 2.2; disabled by default in some releases after an early data-corruption bug. |
| ext4 | No | `--reflink=always` fails; `--reflink=auto` does a full copy. |
| NFS 4.2 | If server FS supports it | Server-side clone. |

### Other OS

| OS | File system | Reflink |
|---|---|---|
| macOS | APFS | Yes (`cp -c`, Finder duplicate) |
| macOS | HFS+ | No |
| Windows | ReFS | Yes (block cloning; Dev Drive, Hyper-V) |
| Windows | NTFS | No |

### Volume-level CoW (related)

LVM thin snapshots, ZFS snapshots, Btrfs subvolume snapshots and cloud disk snapshots (e.g. AWS EBS) apply the same block-sharing principle to a **whole volume** instead of individual files. Same trade-off, coarser unit: you clone a disk rather than a directory, which suits VM-based environments where reflink-per-file isn't available.

## 6. Common mistakes

- **Assuming the clone is a backup.** It shares blocks with the snapshot on the same volume; losing the volume loses both.
- **Cloning across file systems.** `cp --reflink=auto` will silently do a full copy — the job "works" but takes an hour and fills the disk.
- **Deleting live data before validating the source.** Validate first, swap after.
- **Trusting the job's exit status.** Check that the database actually starts and that a known table has rows.
- **Forgetting masking is a property of the snapshot, not the clone.** If the snapshot-building job stops masking, every environment downstream quietly gains real PII.
