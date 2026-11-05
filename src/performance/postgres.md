# Postgres access patterns

Seven dimensions describe any query. Fifteen families cover the combinations that keep recurring. Each family has a standard index and a standard trick. This guide walks them from everyday to rare.

## The one number that matters

An index is a shortcut from a predicate to the few pages that hold matching rows. Every decision below is in service of one check: the pages Postgres touches should be about the number of rows you get back.

<figure class="fig">
<svg viewBox="0 0 640 190" role="img" aria-label="A heap of 16 pages. A sequential scan touches all 16 to return 3 rows. An index scan touches only the 3 pages holding the rows.">
  <text x="0" y="16" class="hd">Table heap (16 pages)</text>
  <text x="0" y="52" class="mu">seq scan</text>
  <g>
    <rect x="80" y="36" width="30" height="24" class="dim"/><rect x="114" y="36" width="30" height="24" class="dim"/><rect x="148" y="36" width="30" height="24" class="dim"/><rect x="182" y="36" width="30" height="24" class="dim"/>
    <rect x="216" y="36" width="30" height="24" class="dim"/><rect x="250" y="36" width="30" height="24" class="dim"/><rect x="284" y="36" width="30" height="24" class="dim"/><rect x="318" y="36" width="30" height="24" class="dim"/>
    <rect x="352" y="36" width="30" height="24" class="dim"/><rect x="386" y="36" width="30" height="24" class="dim"/><rect x="420" y="36" width="30" height="24" class="dim"/><rect x="454" y="36" width="30" height="24" class="dim"/>
    <rect x="488" y="36" width="30" height="24" class="dim"/><rect x="522" y="36" width="30" height="24" class="dim"/><rect x="556" y="36" width="30" height="24" class="dim"/><rect x="590" y="36" width="30" height="24" class="dim"/>
    <rect x="148" y="36" width="30" height="24" class="wa"/><rect x="386" y="36" width="30" height="24" class="wa"/><rect x="556" y="36" width="30" height="24" class="wa"/>
  </g>
  <text x="80" y="78" class="mono">16 pages touched → 3 rows</text>
  <text x="0" y="128" class="mu">index scan</text>
  <g>
    <rect x="80" y="112" width="30" height="24" class="dimo"/><rect x="114" y="112" width="30" height="24" class="dimo"/><rect x="182" y="112" width="30" height="24" class="dimo"/>
    <rect x="216" y="112" width="30" height="24" class="dimo"/><rect x="250" y="112" width="30" height="24" class="dimo"/><rect x="284" y="112" width="30" height="24" class="dimo"/><rect x="318" y="112" width="30" height="24" class="dimo"/>
    <rect x="352" y="112" width="30" height="24" class="dimo"/><rect x="420" y="112" width="30" height="24" class="dimo"/><rect x="454" y="112" width="30" height="24" class="dimo"/>
    <rect x="488" y="112" width="30" height="24" class="dimo"/><rect x="522" y="112" width="30" height="24" class="dimo"/><rect x="590" y="112" width="30" height="24" class="dimo"/>
    <rect x="148" y="112" width="30" height="24" class="ac"/><rect x="386" y="112" width="30" height="24" class="ac"/><rect x="556" y="112" width="30" height="24" class="ac"/>
  </g>
  <text x="80" y="154" class="mono">3 pages touched → 3 rows</text>
  <text x="0" y="182" class="mu">Colored pages hold a matching row. Filled pages were read.</text>
</svg>
<figcaption>The goal of every index: read the pages that hold your rows and nothing else. The ratio of pages read to rows returned is what <code>EXPLAIN (ANALYZE, BUFFERS)</code> shows you.</figcaption>
</figure>

> [!TIP]
> **Pages touched ≈ rows returned.** When the plan reads far more than it returns, the index is wrong, missing, or ignored.

## Layer 1: seven dimensions of any query

Before picking a family, describe the query. Each dimension answers one design question. Here is a typical query with its dimensions marked.

```sql
select id, title, created_at            -- output: a few columns
from posts
where author_id = $1                    -- predicate: point on non-unique column
  and status = 'published'              -- predicate: equality, skewed
order by created_at desc                -- order: index must deliver it
limit 20                                -- limit: top-N, selectivity tiny
```

<div class="dims">
<div><b>Predicate shape</b><span>point on unique · point on non-unique · range · IN/OR · several columns · text pattern · containment · overlap · computed</span><em>Decides index type and column order</em></div>
<div><b>Selectivity</b><span>tiny (1–100 rows) · small (&lt;5%) · large (&gt;10–20%) · skewed by key</span><em>Index vs bitmap vs seq scan</em></div>
<div><b>Order + limit</b><span>none · ORDER BY … LIMIT n · full sort · pagination</span><em>Must the index deliver the order?</em></div>
<div><b>Output</b><span>a few columns · whole rows · an aggregate</span><em>Covering index, or no index and a rollup</em></div>
<div><b>Write profile</b><span>append-only · update-heavy · hot rows · bulk delete</span><em>What each index costs on writes</em></div>
<div><b>Temporal locality</b><span>uniform · recent rows hot · small live subset</span><em>Partial index, BRIN, partitioning</em></div>
<div><b>Frequency × latency</b><span>hot path · interactive · batch or report</span><em>How much write cost the query justifies</em></div>
</div>

> [!TIP]
> **Above roughly 10–20% of the table, Postgres will choose a sequential scan anyway.** Don't build an index for a predicate that matches most rows.

## Layer 2, tier 1: everyday

*Families 1–5. Nearly every table has these.*

Most of an application's load is these five. They are all served by the B-tree, and the only real skill is column order.

### 1. Point lookup

`where id = ?`

| | |
|---|---|
| **Index** | Primary key or unique B-tree. Already there. |
| **Watch** | Nothing. This is the baseline every other family is measured against. |

### 2. Children of a parent

`where parent_id = ?`

| | |
|---|---|
| **Index** | B-tree on the foreign key column. |
| **Technique** | Nested loop join: the planner looks up each parent's children through the index. |

> [!WARNING]
> Postgres does not index foreign keys for you. A missing one makes joins slow and makes `on delete cascade` scan the whole child table.

> [!TIP]
> **Every foreign key gets an index** unless you have measured a reason not to.

### 3. Filtered top-N

`where a = ? and b = ? order by c limit n`

| | |
|---|---|
| **Index** | Composite `(a, b, c)`: the equality columns first, then the sort column. |

<figure class="fig">
<svg viewBox="0 0 640 250" role="img" aria-label="Two composite indexes compared. With equality columns first, the matching entries form one contiguous block already sorted by the sort column, so the scan reads n entries and stops. With the sort column first, matches are scattered through the whole index.">
  <text x="0" y="16" class="hd">(a, b, c) — equality first</text>
  <text x="330" y="16" class="hd">(c, a, b) — sort column first</text>
  <g class="mono">
    <rect x="0" y="28" width="280" height="18" class="dimo"/><text x="8" y="41">a=1 b=1 c=…</text>
    <rect x="0" y="48" width="280" height="18" class="dimo"/><text x="8" y="61">a=1 b=2 c=…</text>
    <rect x="0" y="68" width="280" height="18" class="acs"/><rect x="0" y="68" width="280" height="18" class="acl"/><text x="8" y="81">a=2 b=1 c=3</text>
    <rect x="0" y="88" width="280" height="18" class="acs"/><rect x="0" y="88" width="280" height="18" class="acl"/><text x="8" y="101">a=2 b=1 c=5</text>
    <rect x="0" y="108" width="280" height="18" class="acs"/><rect x="0" y="108" width="280" height="18" class="acl"/><text x="8" y="121">a=2 b=1 c=8</text>
    <rect x="0" y="128" width="280" height="18" class="acs"/><text x="8" y="141">a=2 b=1 c=9</text>
    <rect x="0" y="148" width="280" height="18" class="dimo"/><text x="8" y="161">a=2 b=2 c=…</text>
    <rect x="0" y="168" width="280" height="18" class="dimo"/><text x="8" y="181">a=3 b=1 c=…</text>
    <rect x="0" y="188" width="280" height="18" class="dimo"/><text x="8" y="201">a=3 b=2 c=…</text>
  </g>
  <text x="0" y="230" class="mu">a=2, b=1 → one block, already ordered by c. Read 3, stop.</text>
  <g class="mono">
    <rect x="330" y="28" width="280" height="18" class="acs"/><text x="338" y="41">c=1 a=3 b=1</text>
    <rect x="330" y="48" width="280" height="18" class="acs"/><rect x="330" y="48" width="280" height="18" class="wal"/><text x="338" y="61">c=3 a=2 b=1  ✓</text>
    <rect x="330" y="68" width="280" height="18" class="acs"/><text x="338" y="81">c=4 a=1 b=2</text>
    <rect x="330" y="88" width="280" height="18" class="acs"/><rect x="330" y="88" width="280" height="18" class="wal"/><text x="338" y="101">c=5 a=2 b=1  ✓</text>
    <rect x="330" y="108" width="280" height="18" class="acs"/><text x="338" y="121">c=6 a=3 b=2</text>
    <rect x="330" y="128" width="280" height="18" class="acs"/><text x="338" y="141">c=7 a=1 b=1</text>
    <rect x="330" y="148" width="280" height="18" class="acs"/><rect x="330" y="148" width="280" height="18" class="wal"/><text x="338" y="161">c=8 a=2 b=1  ✓</text>
    <rect x="330" y="168" width="280" height="18" class="acs"/><text x="338" y="181">c=9 a=2 b=2</text>
    <rect x="330" y="188" width="280" height="18" class="acs"/><text x="338" y="201">c=9 a=2 b=1</text>
  </g>
  <text x="330" y="230" class="mu">Walks the index in c order, filtering every entry.</text>
</svg>
<figcaption>Shaded rows are what the scan reads. Equality columns first make the matches contiguous and pre-sorted, so the scan stops after n entries.</figcaption>
</figure>

> [!TIP]
> **Composite column order: equality → sort → range.** Equality columns, then the sort column, then the range column.

> [!WARNING]
> Skewed keys. One author with a million posts behaves differently from one with ten. Test the plan on both a hot key and a cold key.

### 4. Paginated list

Family 3, plus "page N".

| | |
|---|---|
| **Index** | `(…, c, id)`: add a unique tiebreaker to the sort key. |
| **Technique** | Keyset pagination with `where (c, id) < ($last_c, $last_id)`. Fetch `limit n+1` to know if there is a next page. OFFSET only for shallow pages. |

<figure class="fig">
<svg viewBox="0 0 640 170" role="img" aria-label="OFFSET pagination walks and discards every row before the requested page; keyset pagination jumps straight to the last seen key and reads only the page.">
  <defs><marker id="arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="currentColor"/></marker></defs>
  <text x="0" y="16" class="hd">Index, sorted by (c, id)</text>
  <rect x="0" y="28" width="640" height="20" class="dimo"/>
  <line x1="470" y1="28" x2="470" y2="48" class="st"/><line x1="530" y1="28" x2="530" y2="48" class="st"/>
  <text x="500" y="42" text-anchor="middle" class="mono">page 50</text>
  <text x="0" y="82" class="mu">offset 980</text>
  <rect x="80" y="68" width="390" height="18" class="wab"/>
  <line x1="84" y1="77" x2="462" y2="77" class="wal" marker-end="url(#arr)"/>
  <text x="270" y="81" text-anchor="middle" class="mono">read 980 rows, throw them away</text>
  <rect x="470" y="68" width="60" height="18" class="ac"/>
  <text x="0" y="128" class="mu">keyset</text>
  <path d="M84 123 Q 280 90 466 118" class="acl" marker-end="url(#arr)"/>
  <text x="275" y="140" text-anchor="middle" class="mono">seek to (last_c, last_id), read 20</text>
  <rect x="470" y="114" width="60" height="18" class="ac"/>
  <text x="0" y="164" class="mu">Both return the same 20 rows. Only one of them costs the same on page 50 as on page 1.</text>
</svg>
<figcaption>OFFSET cost grows with page depth. Keyset cost is constant because the index seeks directly to the last key seen.</figcaption>
</figure>

> [!TIP]
> **Any sort used for pagination ends in a unique column.** Without the tiebreaker, equal values can repeat or skip across pages.

### 5. Time or number range

`where a = ? and ts between ? and ?`

| | |
|---|---|
| **Index** | `(a, ts)`. For huge append-only tables, a BRIN index on `ts` is tiny and often enough. |
| **Technique** | Partition by time when the table is large and queries are time-bounded. |

<figure class="fig">
<svg viewBox="0 0 640 150" role="img" aria-label="With the equality column first, the range of timestamps for one key is a single contiguous run. With the timestamp first, the key's rows are spread across the whole time range.">
  <text x="0" y="16" class="hd">(a, ts)</text>
  <rect x="0" y="26" width="640" height="18" class="dimo"/>
  <rect x="210" y="26" width="230" height="18" class="acs"/>
  <rect x="290" y="26" width="90" height="18" class="ac"/>
  <text x="0" y="60" class="mono">│ a=1 … │ a=2, ts ascending ▮▮▮ range ▮▮▮ │ a=3 … │</text>
  <text x="0" y="104" class="hd">(ts, a)</text>
  <rect x="0" y="114" width="640" height="18" class="dimo"/>
  <rect x="150" y="114" width="300" height="18" class="acs"/>
  <rect x="160" y="114" width="6" height="18" class="ac"/><rect x="205" y="114" width="6" height="18" class="ac"/><rect x="260" y="114" width="6" height="18" class="ac"/><rect x="300" y="114" width="6" height="18" class="ac"/><rect x="355" y="114" width="6" height="18" class="ac"/><rect x="410" y="114" width="6" height="18" class="ac"/>
  <text x="0" y="146" class="mono">│ all keys in time order; a=2 is scattered through the whole range │</text>
</svg>
<figcaption>Dark marks are the rows wanted; light shading is what gets scanned. A range column before an equality column forces the scan to cover every key inside the range.</figcaption>
</figure>

> [!WARNING]
> A range column placed before an equality column. The index can only seek on columns up to the first range, so everything after it becomes a filter.

## Tier 2: operational

*Families 6–9. Admin screens, queues, invariants, counts.*

These appear in most systems once they have users, and they are where the plain composite B-tree stops being enough. The answers are still cheap, but each needs a specific trick.

### 6. Ad-hoc multi-filter

Any combination of optional wheres (admin search).

| | |
|---|---|
| **Index** | Several single-column indexes. The planner combines them with bitmap scans. |
| **Technique** | Build the SQL from the filters actually set, so unused filters don't appear in the query. |

<figure class="fig">
<svg viewBox="0 0 640 210" role="img" aria-label="Three single-column indexes each produce a bitmap of matching pages. The bitmaps are ANDed, and only pages set in all three are fetched from the heap.">
  <text x="0" y="16" class="hd">BitmapAnd: three single-column indexes, one heap pass</text>
  <g class="mono">
    <text x="0" y="48">idx(status)</text>
    <text x="0" y="78">idx(country)</text>
    <text x="0" y="108">idx(plan)</text>
    <text x="0" y="160">pages fetched</text>
  </g>
  <g>
    <rect x="120" y="36" width="26" height="14" class="ac"/><rect x="150" y="36" width="26" height="14" class="dim"/><rect x="180" y="36" width="26" height="14" class="ac"/><rect x="210" y="36" width="26" height="14" class="ac"/><rect x="240" y="36" width="26" height="14" class="dim"/><rect x="270" y="36" width="26" height="14" class="ac"/><rect x="300" y="36" width="26" height="14" class="ac"/><rect x="330" y="36" width="26" height="14" class="dim"/><rect x="360" y="36" width="26" height="14" class="ac"/><rect x="390" y="36" width="26" height="14" class="dim"/><rect x="420" y="36" width="26" height="14" class="ac"/><rect x="450" y="36" width="26" height="14" class="ac"/><rect x="480" y="36" width="26" height="14" class="dim"/><rect x="510" y="36" width="26" height="14" class="ac"/><rect x="540" y="36" width="26" height="14" class="dim"/><rect x="570" y="36" width="26" height="14" class="ac"/>
    <rect x="120" y="66" width="26" height="14" class="dim"/><rect x="150" y="66" width="26" height="14" class="ac"/><rect x="180" y="66" width="26" height="14" class="ac"/><rect x="210" y="66" width="26" height="14" class="dim"/><rect x="240" y="66" width="26" height="14" class="ac"/><rect x="270" y="66" width="26" height="14" class="ac"/><rect x="300" y="66" width="26" height="14" class="dim"/><rect x="330" y="66" width="26" height="14" class="ac"/><rect x="360" y="66" width="26" height="14" class="ac"/><rect x="390" y="66" width="26" height="14" class="dim"/><rect x="420" y="66" width="26" height="14" class="dim"/><rect x="450" y="66" width="26" height="14" class="ac"/><rect x="480" y="66" width="26" height="14" class="ac"/><rect x="510" y="66" width="26" height="14" class="ac"/><rect x="540" y="66" width="26" height="14" class="dim"/><rect x="570" y="66" width="26" height="14" class="ac"/>
    <rect x="120" y="96" width="26" height="14" class="ac"/><rect x="150" y="96" width="26" height="14" class="dim"/><rect x="180" y="96" width="26" height="14" class="ac"/><rect x="210" y="96" width="26" height="14" class="ac"/><rect x="240" y="96" width="26" height="14" class="ac"/><rect x="270" y="96" width="26" height="14" class="dim"/><rect x="300" y="96" width="26" height="14" class="dim"/><rect x="330" y="96" width="26" height="14" class="dim"/><rect x="360" y="96" width="26" height="14" class="ac"/><rect x="390" y="96" width="26" height="14" class="ac"/><rect x="420" y="96" width="26" height="14" class="dim"/><rect x="450" y="96" width="26" height="14" class="ac"/><rect x="480" y="96" width="26" height="14" class="dim"/><rect x="510" y="96" width="26" height="14" class="dim"/><rect x="540" y="96" width="26" height="14" class="ac"/><rect x="570" y="96" width="26" height="14" class="ac"/>
  </g>
  <text x="358" y="134" text-anchor="middle" class="mono">AND ↓</text>
  <g>
    <rect x="120" y="148" width="26" height="14" class="dimo"/><rect x="150" y="148" width="26" height="14" class="dimo"/><rect x="180" y="148" width="26" height="14" class="ac"/><rect x="210" y="148" width="26" height="14" class="dimo"/><rect x="240" y="148" width="26" height="14" class="dimo"/><rect x="270" y="148" width="26" height="14" class="dimo"/><rect x="300" y="148" width="26" height="14" class="dimo"/><rect x="330" y="148" width="26" height="14" class="dimo"/><rect x="360" y="148" width="26" height="14" class="ac"/><rect x="390" y="148" width="26" height="14" class="dimo"/><rect x="420" y="148" width="26" height="14" class="dimo"/><rect x="450" y="148" width="26" height="14" class="ac"/><rect x="480" y="148" width="26" height="14" class="dimo"/><rect x="510" y="148" width="26" height="14" class="dimo"/><rect x="540" y="148" width="26" height="14" class="dimo"/><rect x="570" y="148" width="26" height="14" class="ac"/>
  </g>
  <text x="0" y="196" class="mu">Each index alone matches many pages. Together they narrow to 4. No composite index needed for this combination.</text>
</svg>
<figcaption>Bitmap scans let independent single-column indexes cooperate. One index per filterable column covers every combination of filters.</figcaption>
</figure>

> [!TIP]
> **Composite indexes for known hot-path queries. Single-column indexes for ad-hoc search.** Don't try to build a composite per filter combination.

### 7. Hot subset / work queue

`where status = 'queued' order by created_at limit 1`

| | |
|---|---|
| **Index** | Partial index: `create index … on jobs (created_at) where status = 'queued'`. |
| **Technique** | `for update skip locked` so workers don't block each other. Archive or partition finished rows. |

<figure class="fig">
<svg viewBox="0 0 640 160" role="img" aria-label="A jobs table where almost every row is done and a handful are queued. A full index covers every row; a partial index covers only the queued rows and stays tiny.">
  <text x="0" y="16" class="hd">jobs table: 1,000,000 done, 40 queued</text>
  <rect x="0" y="28" width="640" height="22" class="dim"/>
  <rect x="96" y="28" width="4" height="22" class="ac"/><rect x="233" y="28" width="4" height="22" class="ac"/><rect x="410" y="28" width="4" height="22" class="ac"/><rect x="598" y="28" width="4" height="22" class="ac"/><rect x="620" y="28" width="4" height="22" class="ac"/>
  <text x="0" y="68" class="mu">full index on (created_at)</text>
  <rect x="200" y="58" width="440" height="12" class="dim"/>
  <text x="0" y="104" class="mu">partial index … where status = 'queued'</text>
  <rect x="280" y="94" width="22" height="12" class="ac"/>
  <text x="310" y="104" class="mono">fits in cache, always</text>
  <text x="0" y="146" class="mu">Dark marks are the queued rows. The partial index only ever contains those.</text>
</svg>
<figcaption>A partial index indexes the live subset only, so a queue of 40 rows stays a 40-entry index no matter how many jobs have finished.</figcaption>
</figure>

> [!WARNING]
> Churn. Every job updates status at least twice, which bloats both table and index. Long-running transactions stop vacuum from reclaiming that space.

> [!TIP]
> **When the query only ever wants a small, well-defined subset, index that subset** with a `where` clause.

### 8. Existence / uniqueness

`exists(…)`, or a business rule such as "no two active users share an email".

| | |
|---|---|
| **Index** | Unique index. Partial unique when the rule only applies to live rows: `unique (email) where deleted_at is null`. |
| **Technique** | `insert … on conflict`. Let the database enforce the rule in one statement. |

> [!WARNING]
> Check-then-insert in application code races under concurrency. Two requests both see "not found" and both insert.

> [!TIP]
> **A business uniqueness rule is a unique index**, never a SELECT followed by an INSERT.

### 9. Count / aggregate

`count(*)`, `sum(amount)` over many rows.

| | |
|---|---|
| **Index** | A covering index at best. It is still O(n) in the rows counted. |
| **Technique** | Counter tables, rollups, materialized views, planner estimates, or capped counts ("more than 1,000"). |

<figure class="fig">
<svg viewBox="0 0 640 150" role="img" aria-label="Counting rows directly costs time proportional to the rows; a counter table is one row read, at the cost of a hot row updated on every write.">
  <defs><marker id="arr2" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="currentColor"/></marker></defs>
  <text x="0" y="16" class="hd">count(*) where author_id = ?</text>
  <text x="0" y="50" class="mu">live count</text>
  <rect x="110" y="38" width="360" height="16" class="acs"/>
  <line x1="114" y1="46" x2="462" y2="46" class="acl" marker-end="url(#arr2)"/>
  <text x="290" y="50" text-anchor="middle" class="mono">walk every matching index entry</text>
  <text x="480" y="50" class="mono">O(n)</text>
  <text x="0" y="98" class="mu">counter table</text>
  <rect x="110" y="86" width="40" height="16" class="ac"/>
  <text x="160" y="98" class="mono">one row</text>
  <text x="480" y="98" class="mono">O(1)</text>
  <text x="110" y="134" class="mu">…but every insert now also updates that one row. Hot row, lock contention.</text>
</svg>
<figcaption>There is no index that makes counting free. You trade read cost for write cost, or you trade exactness for an estimate or a cap.</figcaption>
</figure>

> [!TIP]
> **If the UI only needs "99+", cap the count** with `limit 100` in a subquery. Exact counts over big sets belong in a rollup.

## Tier 3: specialized predicates

*Families 10–13. The B-tree can't do these alone.*

Each of these needs either a different index type or a rewrite of the predicate. The signal that you're here: the predicate isn't `=`, `<`, `>`, or `between` on a plain column.

### 10. Text search

`like 'abc%'` · `ilike '%abc%'` · words and relevance

<figure class="fig">
<svg viewBox="0 0 640 200" role="img" aria-label="Three text predicates and the index each needs. A prefix match works on a B-tree because the sorted entries share the prefix. A substring match needs trigrams. Word search needs a tsvector.">
  <g class="hd"><text x="0" y="16">predicate</text><text x="220" y="16">index</text><text x="400" y="16">why</text></g>
  <line x1="0" y1="24" x2="640" y2="24" class="st"/>
  <g class="mono">
    <text x="0" y="52">like 'abc%'</text>
    <text x="220" y="52">B-tree text_pattern_ops</text>
  </g>
  <g><rect x="400" y="40" width="34" height="14" class="dimo"/><rect x="438" y="40" width="34" height="14" class="ac"/><rect x="476" y="40" width="34" height="14" class="ac"/><rect x="514" y="40" width="34" height="14" class="dimo"/></g>
  <text x="400" y="70" class="mu">sorted: abc… entries are adjacent</text>
  <g class="mono">
    <text x="0" y="108">ilike '%abc%'</text>
    <text x="220" y="108">GIN pg_trgm</text>
  </g>
  <text x="400" y="108" class="mono">"abc" → [ab][abc][bc]</text>
  <text x="400" y="126" class="mu">lookup each trigram, intersect</text>
  <g class="mono">
    <text x="0" y="164">@@ to_tsquery('run &amp; fast')</text>
    <text x="220" y="164">GIN on tsvector</text>
  </g>
  <text x="400" y="164" class="mono">"running" → run</text>
  <text x="400" y="182" class="mu">stems, stop words, rank after</text>
</svg>
<figcaption>Which text predicate you write decides the index. A plain B-tree serves a prefix only; anything starting with a wildcard needs GIN.</figcaption>
</figure>

> [!TIP]
> **Cap the result count before ranking.** Search returns at most a few hundred candidates; relevance sorting happens on that set, never on the whole table.

### 11. Containment (JSON, arrays, tags)

`tags @> '{x}'` · `doc @> '{"k": "v"}'`

| | |
|---|---|
| **Index** | GIN. It indexes every element or key inside the value, so a row appears under many entries. |

> [!WARNING]
> Every write touches one index entry per element. On a write-heavy table a GIN index is expensive to maintain, and its pending-list behaviour makes write latency uneven.

### 12. Overlap / nearest (geo, booking ranges)

`range && range` · `point <-> point` · "no double booking"

| | |
|---|---|
| **Index** | GiST. Sorted order doesn't exist for "overlaps" or "near", so a B-tree can't help. |
| **Technique** | Exclusion constraint: `exclude using gist (room_id with =, during with &&)` makes double booking impossible at the database level. |

> [!TIP]
> **If the predicate can't be expressed as "is this value before or after that one", it isn't a B-tree problem.** Pick the index type from the operator.

### 13. Computed predicate

`where lower(email) = ?` · `where created_at::date = ?`

| | |
|---|---|
| **Index** | Expression index on `lower(email)`, or a generated column with a plain index. |
| **Technique** | Rewrite the predicate as a range the existing index can use: `created_at >= $d and created_at < $d + interval '1 day'`. |

> [!WARNING]
> The index on `created_at` is silently ignored when you wrap the column in a function. No error, just a sequential scan.

> [!TIP]
> **Never wrap the indexed column in a function in a WHERE clause.** Move the computation to the other side of the comparison.

## Tier 4: lifecycle and scale

*Families 14–15. The ones with no index answer.*

These two look like queries but are really data-layout problems. The right move is to change where the data lives, not how it's indexed.

### 14. Retention / bulk delete

`delete from events where ts < now() - interval '90 days'`

| | |
|---|---|
| **Structure** | Partition by time. Dropping a partition is a metadata operation. |
| **Technique** | Without partitions: batched keyset deletes of a few thousand rows at a time. |

<figure class="fig">
<svg viewBox="0 0 640 190" role="img" aria-label="Bulk delete on one table marks millions of rows dead and leaves bloat for vacuum, while replicas replay every row. Dropping a partition removes the whole file at once.">
  <text x="0" y="16" class="hd">single table, delete where ts &lt; cutoff</text>
  <rect x="0" y="26" width="640" height="22" class="dimo"/>
  <rect x="0" y="26" width="300" height="22" class="wab"/>
  <text x="150" y="41" text-anchor="middle" class="mono">millions of rows marked dead → bloat, WAL, replica lag</text>
  <text x="470" y="41" text-anchor="middle" class="mono">live rows</text>
  <text x="0" y="108" class="hd">partitioned by month, drop partition</text>
  <g>
    <rect x="0" y="118" width="100" height="22" class="wab"/><rect x="0" y="118" width="100" height="22" class="wal dash"/>
    <rect x="108" y="118" width="100" height="22" class="wab"/><rect x="108" y="118" width="100" height="22" class="wal dash"/>
    <rect x="216" y="118" width="100" height="22" class="wab"/><rect x="216" y="118" width="100" height="22" class="wal dash"/>
    <rect x="324" y="118" width="100" height="22" class="dimo"/><rect x="432" y="118" width="100" height="22" class="dimo"/><rect x="540" y="118" width="100" height="22" class="dimo"/>
  </g>
  <g class="mono" text-anchor="middle"><text x="50" y="133">jun</text><text x="158" y="133">jul</text><text x="266" y="133">aug</text><text x="374" y="133">sep</text><text x="482" y="133">oct</text><text x="590" y="133">nov</text></g>
  <text x="0" y="170" class="mu">drop table events_2026_06: one file unlinked, no dead rows, nothing for vacuum</text>
</svg>
<figcaption>A row-by-row delete pays for every row twice: once to mark it dead, once for vacuum. Dropping a partition pays nothing per row.</figcaption>
</figure>

> [!WARNING]
> Cascades. A bulk delete on a parent fans out to child tables, each of which needs the foreign key index from family 2 or the delete becomes a scan per parent row.

> [!TIP]
> **If you know at table creation that rows expire, partition by the expiry column from day one.** Retrofitting is the expensive path.

### 15. Analytics / full scan

`select region, sum(revenue) from orders group by region`

| | |
|---|---|
| **Index** | None. A parallel sequential scan is the right plan when most of the table is read. |
| **Technique** | Run it on a read replica, precompute rollup tables, or move the workload to a columnar or OLAP store. |

> [!WARNING]
> Running this on the primary at high request rates. It evicts the hot working set from cache and slows every point lookup behind it.

> [!TIP]
> **A query that reads most of the table doesn't want an index.** It wants to run somewhere else, or less often.

## Layer 3: the three menus every family picks from

Every family above chose from the same three short menus. Knowing them lets you handle a query that isn't quite any family.

### Index structures

<figure class="fig">
<svg viewBox="0 0 640 240" role="img" aria-label="Decision flow from predicate operator to index structure: comparison operators go to B-tree; containment, text and arrays go to GIN; overlap, distance and exclusion go to GiST; huge append-only ordered data goes to BRIN; vector similarity goes to HNSW.">
  <defs><marker id="arr3" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="currentColor"/></marker></defs>
  <rect x="0" y="92" width="150" height="48" rx="3" class="acs"/><rect x="0" y="92" width="150" height="48" rx="3" class="acl"/>
  <text x="75" y="112" text-anchor="middle" class="hd">what's the operator?</text>
  <text x="75" y="129" text-anchor="middle" class="mu">on the indexed column</text>
  <g class="st"><line x1="150" y1="116" x2="240" y2="26" marker-end="url(#arr3)"/><line x1="150" y1="116" x2="240" y2="72" marker-end="url(#arr3)"/><line x1="150" y1="116" x2="240" y2="116" marker-end="url(#arr3)"/><line x1="150" y1="116" x2="240" y2="160" marker-end="url(#arr3)"/><line x1="150" y1="116" x2="240" y2="206" marker-end="url(#arr3)"/></g>
  <g class="mono">
    <text x="250" y="30">= &lt; &gt; between, like 'x%'</text>
    <text x="250" y="76">@&gt; ? &amp;&amp;(array) '%x%' @@</text>
    <text x="250" y="120">&amp;&amp;(range) &lt;-&gt; exclude</text>
    <text x="250" y="164">range on huge append-only</text>
    <text x="250" y="210">&lt;=&gt; vector similarity</text>
  </g>
  <g class="hd" text-anchor="end">
    <text x="640" y="30">B-tree</text>
    <text x="640" y="76">GIN</text>
    <text x="640" y="120">GiST</text>
    <text x="640" y="164">BRIN</text>
    <text x="640" y="210">HNSW / IVFFlat</text>
  </g>
  <g class="mu" text-anchor="end">
    <text x="640" y="46">single · composite · partial · covering · expression · unique</text>
    <text x="640" y="92">JSONB, arrays, full-text, trigram</text>
    <text x="640" y="136">ranges, geo, nearest, exclusion</text>
    <text x="640" y="180">tiny index, physical order must match</text>
    <text x="640" y="226">pgvector</text>
  </g>
</svg>
<figcaption>The operator in the WHERE clause picks the structure. B-tree covers nearly everything; the rest exist for operators that have no sort order. Hash exists too, equality only, and is rarely better than B-tree.</figcaption>
</figure>

> [!TIP]
> **Start with a B-tree.** Reach for another type only when the operator isn't a comparison.

### Data layout

- **Partitioning** for retention, time windows, big queues.
- **CLUSTER** to put rows in index order on disk, applied once.
- **fillfactor** below 100 so updates stay on the same page (HOT updates).
- **Denormalized columns and counter tables** when a join or count is on the hot path.
- **Materialized views and rollups** for aggregates read far more than written.
- **Archive tables** for rows that must exist but are never queried in the hot path.

### Query techniques

- **Keyset pagination**, `limit n+1`, capped counts, deferred join.
- **Batching by primary key** for any bulk write.
- **`skip locked`** for queues.
- **`UNION ALL`** instead of an OR across different columns.
- **`LATERAL`** for top-N per group: `cross join lateral (select … where x = outer.x order by … limit n)` runs one indexed family-3 lookup per outer row, instead of numbering every row with a window function.
- **`EXISTS`** instead of JOIN + DISTINCT.
- **Aggregate before joining**, not after.

## How to use it

1. **Describe the query on the seven dimensions.** Predicate shape and selectivity first; they decide the most.
2. **Find the family.** Real queries are often two combined: 3 + 4 is a paginated filtered list, 7 + 14 is a queue that also needs retention. Take the index from one and the technique from the other.
3. **Choose from the menus, then verify.** Run `EXPLAIN (ANALYZE, BUFFERS)` on a hot key and a cold key. Shared buffers hit plus read should be about the rows returned.

```sql
explain (analyze, buffers)
select … where author_id = $hot_key order by created_at desc limit 20;

-- look for:
--   Index Scan using posts_author_created_idx   (not Seq Scan, not Sort)
--   rows=20                                      (not rows=20 after Filter removed 50,000)
--   Buffers: shared hit=24                       (≈ rows, not thousands)
```

Why 24 is good for 20 rows: Postgres stores everything in 8 KB pages, and the Buffers line counts the pages touched. An index scan walks the B-tree from root to leaf (3 to 4 pages) and then fetches up to one heap page per returned row (20). About 24 is therefore the floor for this query. Thousands would mean the plan read and discarded far more than it returned, even if the node still says "Index Scan" and the timing still looks fine on a warm cache.

> [!TIP]
> **This guide describes one query at a time.** One index serving several queries, write cost, and growth are judgment calls made by weighing families against each other.
