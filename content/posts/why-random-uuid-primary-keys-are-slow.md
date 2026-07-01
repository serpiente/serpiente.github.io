+++
title = "Why a random UUID primary key is slow (and UUIDv7 fixes it)"
date = "2026-07-01"
description = "A Postgres 15 benchmark: sequential bigint vs UUIDv4 vs UUIDv7 as a primary key. Measuring insert time, WAL generated, index bloat — and reading the CPU flamegraphs to see exactly where the time goes."
+++

**TLDR:** a random `UUIDv4` primary key makes `INSERT`s slower, generates far more WAL, and bloats the index — not because UUIDs are "big", but because every random key lands in a *different, uncached* page of the B-tree. A sequential `bigint` or an *ordered* `UUIDv7` keeps every insert on the same hot page. I built a small, isolated Postgres 15 POC to measure it, then profiled the backend to see it in the CPU itself.

---

## The setup

Three tables, identical in every way **except the primary key**:

```sql
CREATE TABLE t_seq (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, ...);  -- sequential
CREATE TABLE t_v4  (id uuid PRIMARY KEY DEFAULT gen_random_uuid(),        ...); -- random UUIDv4
CREATE TABLE t_v7  (id uuid PRIMARY KEY DEFAULT uuid_generate_v7(),       ...); -- ordered UUIDv7
```

Postgres 15 doesn't ship a native `uuidv7()` (that arrived in PG18), so the POC implements one in pure SQL: 48 bits of millisecond timestamp, then version/variant bits, then random — so the *high bytes are time-ordered*, which is the whole point.

```sql
CREATE OR REPLACE FUNCTION uuid_generate_v7() RETURNS uuid AS $$
DECLARE ts_ms bytea; rnd bytea; b bytea;
BEGIN
  ts_ms := substring(int8send((extract(epoch FROM clock_timestamp())*1000)::bigint) FROM 3);
  rnd   := substring(uuid_send(gen_random_uuid()) FROM 7 FOR 10);
  b := ts_ms || rnd;
  b := set_byte(b, 6, (get_byte(b, 6) & 15) | 112);  -- version 0111
  b := set_byte(b, 8, (get_byte(b, 8) & 63) | 128);  -- variant 10
  RETURN encode(b, 'hex')::uuid;
END $$ LANGUAGE plpgsql VOLATILE;
```

The Postgres container is deliberately configured to make the effect visible and realistic:

- `shared_buffers = 64MB` — small on purpose, so the random index **doesn't fit in cache** and you get real eviction pressure.
- `checkpoint_timeout = 30min`, `max_wal_size = 8GB` — spaced-out checkpoints, so `full_page_writes` show up in the WAL the way they do under real continuous load.

To keep the comparison honest, the improved run pre-generates all the IDs into a staging table *before* the timer starts (so we measure the insert into table + index, not the cost of generating the UUID), and inserts in batches with a `CHECKPOINT` between each batch — mimicking continuous load punctuated by periodic checkpoints.

## What we're measuring

Three things, for the same number of rows:

1. **Insert time**
2. **WAL bytes generated** — captured with `pg_current_wal_lsn()` / `pg_wal_lsn_diff()` around the insert. This is the star metric.
3. **Final index size** — bloat from page splits.

The mechanism, before the numbers: a B-tree stays cheap when new keys append to the *rightmost* page — that page is hot in cache, the insert touches one buffer, and the WAL record is small. That's exactly what a sequential `bigint` (and an ordered `UUIDv7`) does. A random `UUIDv4` scatters keys across the whole key space: each insert targets a *different* leaf page, which is almost never in `shared_buffers`, so Postgres has to read it from disk, dirty it, and — right after a checkpoint — write the entire 8KB page into the WAL as a full-page image. More page splits, more bloat, more WAL, more I/O.

## Reading it in the CPU

Numbers tell you *that* it's slower; a flamegraph tells you *why*. I profiled the inserting backend with `perf` (native `arm64`, so the stacks are clean) during an 8M-row insert, for the sequential key and the random `UUIDv4` key. Same insert, same table shape — only the primary key differs.

The asymmetry is stark. Here's where the on-CPU time goes (inclusive %, sequential → v4):

| Function group | Sequential | UUIDv4 |
|---|---|---|
| `_bt_*` (B-tree descent + insert) | 24.6% | **77.9%** |
| buffer manager (find/evict pages) | 13.8% | **58.8%** |
| `_bt_compare` (compare keys down the tree) | 2.0% | 12.5% |
| `heap_insert` (write the row) | 20.3% | 5.0% |
| `XLogInsert` + CRC (build WAL records) | 16.1% | 4.2% |
| generate UUID / random | 26.5% | 11.5% |

With a sequential key the work is spread sensibly across writing the heap row, building WAL, and generating the value. With `UUIDv4`, the backend **lives inside the B-tree and the buffer manager** — nearly 60% of CPU is spent finding and evicting index pages that aren't in cache. The index tail wags the whole insert.

The interactive comparison below shows both flamegraphs side by side — hover any block, or search for a function (try `_bt_`, `buffer`, or `ReadBuffer`) to highlight the same branch in both profiles and see, literally, how much wider it gets when the key is random:

{{ iframe(src="/uuid-flamegraphs.html") }}

## Takeaways

- **The cost of a random PK isn't the 16 bytes.** It's the *access pattern*: random keys destroy B-tree and buffer-cache locality. That shows up as I/O, WAL, and index bloat — not as "UUIDs are bigger."
- **UUIDv7 gets you the best of both.** You keep globally-unique, client-generatable, non-guessable IDs *and* the insert locality of a sequential key, because the timestamp prefix keeps new keys clustered at the right edge of the index.
- **If you're on PG15**, you can generate v7 in pure SQL today (as above); PG18 ships it natively as `uuidv7()`.
- **If you already have a `UUIDv4` PK** and inserts are your bottleneck, this is the thing to look at before you reach for bigger hardware.

The full POC — `docker compose up`, run the benchmark, generate your own flamegraphs — is a handful of small files: a `docker-compose.yml`, the benchmark SQL, and a `perf` profiling script.
