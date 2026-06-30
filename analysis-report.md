# Root Cause Analysis — Stage Server `stg_TakeoffPrelive`

**Scope:** Diagnostic analysis ONLY — symptoms and root causes. No `CREATE INDEX`, no
rewrites, no fixes in this document (those ship separately as scripts).
**Date:** 2026-06-30
**Database:** `stg_TakeoffPrelive` (one of several tenant Prelive DBs on the same server)
**Artifacts used:** `procedures/`, `execution-plans/` (3 `.sqlplan` + 2 captured-plan findings
notes), `stats/` (`execution-stats-all-procs.txt`, `stats-freshness.txt`, `column-types.txt`,
`table-rowcounts.txt`, `table-indexes.txt`, `missing-indexes.txt`, `statistics-io-time.txt`).

> **Confidence note:** All four procedures now have BOTH aggregated runtime stats
> (`sys.dm_exec_procedure_stats`) AND an actual execution plan. Earlier gaps
> (`SP_ADMIN` / `TicketList` had no plan; `GetFilteredTickets` / `TicketList` had no
> procedure stats) are CLOSED — see §8. Findings below are plan-confirmed unless stated.

---

## 0. Executive summary

All four procedures share the **same root-cause family**, which is why all four are slow on
**every tenant database** (not just Takeoff — see §6.7), pointing at shared procedure/schema
design rather than tenant-specific data:

1. **Catch-all optional-parameter predicates** (`col = @p OR @p IS NULL`, `IIF(@p=0,col,@p)`)
   → non-sargable, no base-table index seeks, high parameter-sniffing exposure.
2. **"Materialize / rank the whole table" temp-table builds** that do not push down available
   filters → full scans into tempdb.
3. **`nvarchar` vs `varchar` implicit conversions** on join keys and parameters → the plan
   itself flags "Seek Plan" / "Cardinality Estimate" convert warnings.
4. **`STRING_SPLIT` / `STRING_AGG` / leading-wildcard `LIKE`** → fixed-row cardinality
   guesses and non-sargable predicates.

| Rank | Procedure | Execs | Avg CPU | Avg elapsed | Avg logical reads |
|---|---|---|---|---|---|
| 1 | **TicketList_GetTicketingInfoByStatus** | 2 | **323 ms** | 240 ms | 11,786 |
| 2 | **GetFilteredTickets** | 1 | 233 ms | 219 ms | **15,094** |
| 3 | **AgentAccountLadgerRpt_For_NewSubPurposeType** | 53 | 164 ms | 171 ms | 3,767 |
| 4 | **SP_ADMIN_BOOKING_LIST_STATUS** | 10 | 24 ms | 24 ms | 1,940 |

(Source: `execution-stats-all-procs.txt`, `stg_TakeoffPrelive` rows. TicketList avg CPU 323 ms
> elapsed 240 ms ⇒ it goes **parallel**.)

---

## 1. `dbo.AgentAccountLadgerRpt_For_NewSubPurposeType`

### 1.1 Cost (`dm_exec_procedure_stats`)
| Metric | Value |
|---|---|
| Execution count | 53 |
| Avg CPU (worker) | 163.76 ms |
| Avg elapsed | 171.17 ms |
| Avg logical reads | 3,767 |
| Total logical reads | 199,699 |

Highest call frequency of the four. STATISTICS IO for one run: `BookingInfo` 122 reads,
`AgentBalanceLog` 107 reads; statement total ~83 ms CPU / 114 ms elapsed — the read average is
dominated by the `#UniqueBookingInfo` build.

### 1.2 Top 3 most expensive operations (`plan_dbo_AgentLedger.sqlplan`)
1. **`Table Insert` into `#UniqueBookingInfo` — subtree cost 2.61 (dominant).** De-dups the
   **entire `BookingInfo` table**.
2. **`Sort` + `Segment` + `Sequence Project` (ROW_NUMBER) over all 16,736 `BookingInfo` rows —
   cost ~1.20.** Feeds operator #1.
3. **`Index Scan` of `BookingInfo` (0.11)** plus a **`Table Scan` of `#UniqueBookingInfo`
   (16,258 rows, 0.095)** on read-back.

### 1.3 Root causes
- **`#UniqueBookingInfo` CTE reads and ranks 100% of `BookingInfo` with no predicate**
  (`ROW_NUMBER() OVER (PARTITION BY UniqueTransId ORDER BY id)`, keep `rn=1`). The ledger date
  range filters `AgentBalanceLog`, but that filter is **not propagated** to this booking
  de-dup → every call materializes and sorts the whole table. Single biggest cost.
- **`PlanAffectingConvert` warning — `TRY_CAST(STRING_SPLIT.[value] AS int)`** in the
  `@AssociatedPurposeIds` filter. `STRING_SPLIT` returns a fixed-row estimate (the 7.07-row TVF
  guess in the plan) → the join cannot be sized.
- **Catch-all optional predicates** (`bl.AgentId = @AgentId OR @AgentId IS NULL`, same for
  `@BalanceType`, `@TnxNumber`, `@PNR`, `@TicketNumbers`) → non-sargable, parameter-sniffing
  exposure (§6.1).
- **Final SELECT** joins the temp table to `Users`, `#UniqueBookingInfo`, `TicketInfo`,
  `Invoice`, `SubPurposeType` with an **OR-of-four-tables PNR predicate** — cheap now, scaling
  risk later.

### 1.4 Index / sargability / stats
- **Indexes used:** base access is an **Index Scan** of `BookingInfo` (not a seek) plus
  clustered seeks on small dimensions. The `AgentBalanceLog` composite index
  `IX_AgentBalanceLog_IsRemoved_Status_BalanceType_IsInvoiceWithPay_StatusDate_WithInclude`
  matches the WHERE and is used — healthy.
- **Plan missing-index warning Impact 81.85 is on the `#UniqueBookingInfo` TEMP table** — a
  symptom of the rank-everything pattern, not a base-table fix.
- **Parameter sniffing: HIGH** (13 optional params, all catch-all).

---

## 2. `dbo.GetFilteredTickets` — heaviest by logical reads

### 2.1 Cost
| Metric | Value |
|---|---|
| Execution count (this DB) | 1 |
| Avg CPU | 232.63 ms |
| Avg elapsed | 218.97 ms |
| Avg logical reads | **15,094** |

Per `statistics-io-time.txt` the final SELECT statement is **702 ms CPU / 644 ms elapsed**, with
**`BookingInfo` Scan count 5, logical reads 6,640** — the single most expensive statement
captured. Compile time 401–403 ms.

### 2.2 Top 3 most expensive operations (`plan_dbo_GetFilteredTickets.sqlplan`)
1. **`Parallelism (Gather Streams)` → `Table Insert` into `#Temp_Bookinginfo` — subtree cost
   24.40 (dominates).** The query goes **parallel** just to populate a temp table.
2. **`Clustered Index Scan` of `BookingInfo` — cost 4.66, ~10,820 rows** feeding that insert.
3. **`Table Scan` of `#Temp_Bookinginfo` (11,497 rows)** + a `Distinct Sort`/`Top` + the
   `SegmentInfo` correlated `MIN(ID)` subquery (72 scans).

### 2.3 Root causes
- **`#Temp_Bookinginfo` is built with `STATUS='Confirmed'` only — NO date filter.** The ticket
  temp table gets the date predicate, but the booking temp build does not → even a single-day
  query materializes ~10.8k bookings (this is why it goes parallel and `BookingInfo` shows
  6,640 reads).
- **Non-sargable date predicate — `CONVERT(DATE, ti.CreatedDate) BETWEEN …`** (twice). Wrapping
  the **column** in `CONVERT(DATE,…)` blocks any `CreatedDate` index seek.
- **`PlanAffectingConvert` — `CONVERT_IMPLICIT(nvarchar(max), [bi].[AirlinePNRs])`.** Confirmed
  cause (see §6.6): `TicketInfo.AirlinePNR` is `nvarchar(MAX)`, `BookingInfo.AirlinePNRs` is
  `varchar(500)`, and `COALESCE(ti.AirlinePNR, bi.AirlinePNRs)` forces the varchar to
  `nvarchar(max)`.
- **Leading-wildcard `LIKE '%' + @RefundStatus + '%'`** — non-sargable.
- **Catch-all optional predicates** on `@AgentId`/`@PNR`/`@UniqueTransID`/`@RefundStatus`.
- **Correlated scalar subquery in the JOIN** — `si.Id = (SELECT MIN(ID) FROM SegmentInfo WHERE
  BookingInfoId = bi.Id)`, re-evaluated per booking.
- **`SELECT DISTINCT`** over a wide column list forces a `Distinct Sort` to mask join fan-out.
- **0 base-table index seeks** — the plan has 26 Index Scans, 0 Index Seeks.

### 2.4 Index / missing indexes
- **Plan missing-index Impact 78.71 on `BookingInfo`** — `EQUALITY [Status]`, `INCLUDE`
  (UniqueTransID, PNR, AirlinePNRs, BookingType, JourneyType, PlatingCarrier, PassengerInfoIDs,
  CreatedDate, AgentId). The optimizer wants a covering index for the `#Temp_Bookinginfo` build.
- The other plan suggestion (Impact 13.58) is on the `#Temp_Bookinginfo` TEMP table — symptom.
- Existing `IX_BookingInfo_Status_CreatedDate` is **misleadingly named** — its key is
  `PNR, PlatingCarrier, UniqueTransID, AirlinePNRs, Status` (Status is the 5th key; CreatedDate
  is not a key) → it cannot serve a `Status='Confirmed'` seek.
- **Parameter sniffing: HIGH** (9 optional params; 400 ms compile each cold compile).

---

## 3. `dbo.SP_ADMIN_BOOKING_LIST_STATUS` — now plan-confirmed

Plan captured 2026-06-30; details in `execution-plans/plan_SP_ADMIN_findings.md`. The proc runs
5 statements (4 temp-table builds + 1 final SELECT).

### 3.1 Cost
| Metric | Value |
|---|---|
| Execution count | 10 |
| Avg CPU | 24.28 ms |
| Avg elapsed | 24.35 ms |
| Avg logical reads | 1,940 |

Currently the cheapest of the four — but the plan shape is the same structural risk as the others.

### 3.2 Per-statement cost
| Stmt | Builds | SubtreeCost | Issue |
|---|---|---|---|
| 1 | `SELECT * INTO #Tmp_BookingInfo` | 0.084 | 3 implicit-convert "Seek Plan" warnings |
| 2 | `#Temp_AgentInfo` | 0.088 | trivial scan (828 rows) |
| 3 | `#Temp_UserInfo` (RoleId) | 0.341 | Clustered Index **Scan** of 2,662 Users; missing index 62.80 |
| 4 | `#Temp_Segment` (CreatedDate range) | **0.673** | **most expensive**; SegmentInfo full scan; missing index 73.61 |
| 5 | final SELECT (STRING_AGG/STRING_SPLIT) | 0.063 | cheap only because 1 booking matched test params |

### 3.3 Root causes (all plan-confirmed)
- **Implicit conversion is actively degrading the access path (Stmt 1).** The plan emits three
  `ConvertIssue="Seek Plan"` warnings:
  `CONVERT_IMPLICIT(nvarchar(20),[BI].[Status])=[@Status]`,
  `CONVERT_IMPLICIT(nvarchar(100),[BI].[PNR])=[@PNR]`,
  `CONVERT_IMPLICIT(nvarchar(50),[BI].[UniqueTransID])=[@UniqueTransID]`. The parameters are
  `NVARCHAR`, the columns are `varchar` — SQL Server converts the **column** and flags the
  resulting plan as degraded.
- **Parameter sniffing proven (Stmt 1 ParameterList):** compiled for `@FromDate=@ToDate='2026-05-18'`
  but executed for `2023-11-30 … 2023-12-06`. One sniffed plan serves very different ranges.
- **`SELECT * INTO #Tmp_BookingInfo`** — pulls every (wide) BookingInfo column into tempdb.
- **`#Temp_Segment` build (Stmt 4) scans all of `SegmentInfo`** to keep ~152 rows
  (ActualLogicalReads 155 + 151 clustered lookups); the CreatedDate predicate also compares
  against a `datetimeoffset(7)` constant (another implicit convert). **Missing index Impact
  73.61** on `SegmentInfo(CreatedDate)` INCLUDE(BookingInfoId, Departure, Arrival, Origin,
  Destination, IsRemoved, GroupName).
- **`#Temp_UserInfo` build (Stmt 3) clustered-scans all 2,662 Users** for `RoleId=@RoleId`.
  **Missing index Impact 62.80** on `Users(RoleId)` INCLUDE(Code).
- **Non-sargable catch-alls everywhere** — `IIF(@AgentId=0, BI.AgentId, @AgentId)` (column
  compared to an expression containing the column), `OR @PNR=''`, `BookingType NOT IN ('Manual')`,
  plus `STRING_SPLIT` in a join and two correlated `STRING_AGG` subqueries.
- **`SET QUERY_GOVERNOR_COST_LIMIT 0`** explicitly disables the cost guardrail — a sign the proc
  has tripped cost limits before.
- **Parameter sniffing: HIGH** (13 params, `goto`-branched).

---

## 4. `dbo.TicketList_GetTicketingInfoByStatus` — slowest by CPU, now plan-confirmed

Plan captured 2026-06-30; details in `execution-plans/plan_dbo_TicketList_findings.md`. Largest,
most complex proc (397 lines, branch on `@Status`, `UNION ALL` in the `Ordered` branch).

### 4.1 Cost
| Metric | Value |
|---|---|
| Execution count | 2 |
| Avg CPU | **323.00 ms** |
| Avg elapsed | 239.83 ms |
| Avg logical reads | 11,786 |

CPU > elapsed ⇒ runs parallel. (Across tenants this proc is run far more — e.g. Firsttrip 145
execs avg 188 ms, Triplover 32 execs avg 313 ms.)

> The captured plan used a sparse date range, so its runtime cost looks low; the plan **shapes**
> below are what drive the 323 ms / 11,786-read average on real workloads.

### 4.2 Root causes (plan-confirmed)
- **`#Temp_Segment_Min` ranks the WHOLE `SegmentInfo` (25,137 rows)** with
  `ROW_NUMBER() OVER (PARTITION BY BookingInfoId)` and no filter (statement subtree cost 0.664).
  It Index-Scans `IX_SegmentInfo_BookingInfoId` then does a **per-row Clustered Index Key Lookup**
  (825 logical reads of pure lookups) because the existing index does not cover
  Origin/Destination/Departure/Arrival. (`IX_SegmentInfo_BookingInfoId_Id` includes
  Origin/Destination but **not Departure/Arrival**.)
- **`AirlinePNRs` implicit conversion (final SELECT)** — `PlanAffectingConvert
  CONVERT_IMPLICIT(nvarchar(max),[BI].[AirlinePNRs])`, identical to GetTicketing §2.3. A second
  convert appears in `COALESCE(ti.RefundStatus, ti.VoidStatus)` (varchar vs nvarchar(200)).
- **Very wide rows through a `Table Spool` for the window aggregates** — `SUM(...) OVER()` /
  `COUNT(*) OVER()` feed a Lazy Spool with **AvgRowSize ~18,700 bytes (~18 KB/row)** because 50+
  wide columns are carried through Top/Sort/Spool. This is the operator that balloons
  memory/tempdb as rows grow — prime suspect for the real-world cost.
- **Correlated subquery per ticket** — `ISNULL((SELECT SUM(ReissueCharge) FROM
  TicketReissuePanaltyBreakdown WHERE TicketId=ti.Id),0)`, appearing both in the row value AND
  inside the windowed `SUM`.
- **`CreditNote` join hits the nvarchar mismatch** — `cn.UniqueTransID` is `nvarchar(1000)` vs
  `varchar` elsewhere (§6.6).
- **Non-sargable optional predicates + leading-wildcard `LIKE` on `RefundStatus`** + `NOT IN
  (STRING_SPLIT(IIF(...)))` for `BookingType`.
- **`Ordered` branch (not in captured run)**: `UNION ALL` of two ~70-column queries with
  double-nested `STRING_SPLIT` and an OR'd `EXISTS` against `[HLD].[HoldBalanceInfo]`
  (`hbi.Reference = bi.PNR OR hbi.Reference = bi.UniqueTransID`) — non-seekable join.
- **Parameter sniffing: HIGH** (15 params, branch-dependent).

---

## 5. `BTM.TicketList_GetTicketingInfoByStatus`

`plan_BTM_TicketList.sqlplan` shows a **trivially cheap plan** (max operator subtree cost 0.0197,
no Sort/Spill/missing-index/convert warnings) **only because the `BTM.*` tables are empty (0
rows)** (`table-rowcounts.txt`). The captured plan is **not representative**. The query shape is
comparatively clean (date range with `CONVERT(DATE,…)` on the **parameter** side; simple
`@p=0 OR col=@p` catch-alls). **Re-capture once `BTM` tables hold representative data.**

---

## 6. Cross-cutting findings

### 6.1 Anti-pattern #1 — catch-all optional-parameter predicates (all 4 procs)
`col = @p OR @p IS NULL` / `@p='' OR col=@p` / `IIF(@p=0,col,@p)` everywhere. Non-sargable →
the two heavy plans show **0 base-table index seeks** (26 and 18 Index Scans). One sniffed plan
fits all parameter sets ⇒ **HIGH parameter-sniffing exposure across all four**. Proven by the
SP_ADMIN ParameterList (compiled 2026-05-18, executed 2023-11-30).

### 6.2 Anti-pattern #2 — "rank / materialize the whole table" into tempdb
- AgentLedger: `#UniqueBookingInfo` = ROW_NUMBER over **all** `BookingInfo` (plan #1 cost).
- GetFilteredTickets: `#Temp_Bookinginfo` = all `Status='Confirmed'` bookings, **no date filter**
  (plan #1 cost, forced parallelism).
- TicketList: `#Temp_Segment_Min` = ROW_NUMBER over **all** `SegmentInfo` (25k rows) + Key Lookup.
- SP_ADMIN: `SELECT * INTO #Tmp_BookingInfo`, and `#Temp_Segment` full-scans SegmentInfo.
In each case a filter that exists elsewhere is **not pushed into the materialization step**.

### 6.3 Anti-pattern #3 — non-sargable date handling
`GetFilteredTickets` wraps the **column** in `CONVERT(DATE, ti.CreatedDate)` (twice). The others
convert the **parameter** side (correct). SP_ADMIN's `#Temp_Segment` build also compares
`CreatedDate` against a `datetimeoffset(7)` constant (implicit convert) — partially blocks the seek.

### 6.4 Anti-pattern #4 — `STRING_SPLIT` / `STRING_AGG` → cardinality guesses
AgentLedger (`TRY_CAST(STRING_SPLIT.value AS int)`), SP_ADMIN (`STRING_SPLIT` in a join +
`STRING_AGG` ×2), TicketList (`NOT IN (STRING_SPLIT(…))` + double-nested `STRING_SPLIT`). Fixed
row estimates poison downstream join sizing.

### 6.5 Anti-pattern #5 — leading-wildcard `LIKE '%' + @p + '%'`
GetFilteredTickets and TicketList both, on `RefundStatus`. Always a scan when the param is set.

### 6.6 Anti-pattern #6 — `nvarchar`/`varchar` implicit conversions (CONFIRMED with exact types)
From `stats/column-types.txt`:
- **`CreditNote.UniqueTransID` = `nvarchar(1000)`, `CreditNote.PNR` = `nvarchar(60)`** while the
  same columns are `varchar(50)` / `varchar(100)` everywhere else. `GetFilteredTickets`,
  `TicketList`, and `AgentLedger` all `LEFT JOIN CreditNote … ON cn.UniqueTransID = …` → the
  varchar side is converted to nvarchar at runtime, which can disable an index seek on CreditNote.
- **`TicketInfo.AirlinePNR` = `nvarchar(MAX)` vs `BookingInfo.AirlinePNRs` = `varchar(500)`** →
  `COALESCE(...)` produces the `CONVERT_IMPLICIT(nvarchar(max), AirlinePNRs)` plan warning seen
  in BOTH GetFilteredTickets and TicketList.
- **`NVARCHAR` parameters vs `varchar` columns** — `GetFilteredTickets` (`@PNR`,`@UniqueTransID`
  NVARCHAR) and `SP_ADMIN` (`@Status`,`@PNR` NVARCHAR, `@UniqueTransID` NVARCHAR(max)) → the
  three "Seek Plan" convert warnings proven in the SP_ADMIN plan. (`dbo.TicketList` params are
  `VARCHAR` — no mismatch there.)
- **`BookingInfo.PassengerInfoIDs` = `varchar(MAX)`** — relevant to index design (a MAX column
  should not be an INCLUDE).

### 6.7 Tables most involved in expensive operations
| Table | Rows | Where it hurts |
|---|---|---|
| **`BookingInfo`** | 16,736 | Full scans / parallel temp loads in GetFilteredTickets (6,640 reads), AgentLedger, SP_ADMIN, TicketList. **#1 hotspot.** |
| **`SegmentInfo`** | 25,116 | Whole-table ROW_NUMBER + Key Lookup (TicketList); full CreatedDate scan (SP_ADMIN); correlated MIN(ID) (GetFilteredTickets). |
| **`TicketInfo`** | 18,397 | Correlated EXISTS/NOT EXISTS; **over-indexed (14 NCIs)** (§6.8). |
| **`CreditNote`** | 1,861 | nvarchar(1000)/nvarchar(60) join keys → implicit conversion in 3 procs. |
| **`BookingFareBreakdown`** | 14,847 | Repeated lookups; no index on `BookingId`. |

**Multi-tenant signal:** the same four procedures are slow on **all** tenant DBs
(`execution-stats-all-procs.txt`: Triplover/Firsttrip/Travelchamp/Taketrip), confirming the
cause is the shared procedure/schema design, not Takeoff-specific data.

### 6.8 Existing indexes vs. what queries need
- **`TicketInfo` is over-indexed: 14 nonclustered indexes**, several redundant
  (`IX_TicketInfo_TicketNumbers` vs `NCI-TicketNum`; 3 PassengerName indexes;
  `IX_TicketInfo_Status_IsCurrent_CreatedDate` whose key is actually
  `TicketNumbers,UniqueTransID,Status`). Write amplification with little read benefit.
- **`BookingInfo` index names mislead** — `IX_BookingInfo_Status_CreatedDate` keys
  `PNR,PlatingCarrier,UniqueTransID,AirlinePNRs,Status` (no usable Status-first seek, no
  CreatedDate key), which is why the optimizer raises the 78.71-impact request.
- **No index on `BookingFareBreakdown(BookingId)`** (DMV top hit 97.47%, 11,038 seeks) — the
  `NCI-BkngIDPrcssType` index keys only `PassengerType` despite its name.
- **`SegmentInfo`** has `IX_SegmentInfo_BookingInfoId_Id` INCLUDE(Origin,Destination) but **not
  Departure,Arrival** → the TicketList Key Lookup; and no `CreatedDate`-keyed covering index →
  the SP_ADMIN full scan.
- **`Users`** has only `IX_Users(IsRemoved)` → no `RoleId` index → the SP_ADMIN clustered scan.
- **`RouteInfo`** has 4 overlapping `BookingInfoId` indexes (redundant).
- `missing-indexes.txt` (DMV) is dominated by overlapping `BookingInfo` / `BookingFareBreakdown`
  / `AgentBalanceLog` variants — **must be de-duplicated, not applied verbatim**.

### 6.9 Are statistics up to date?
**Acceptable — not a primary root cause** (`stats/stats-freshness.txt`). Index-level statistics
were refreshed 2025-10-13 with low drift (404–1,531 row changes). Several clustered/auto-column
stats date to 2025-06…2025-08 with 1,200–2,925 changes since (~8–23% drift) — aging but still
under SQL Server's auto-update threshold (~SQRT(1000×rows) ≈ 3,900–5,000), which is why they
haven't auto-refreshed. The bad cardinality estimates in the plans come from
conversions / `STRING_SPLIT` guesses, **not** stale histograms. A one-time
`UPDATE STATISTICS … WITH FULLSCAN` on the five hot tables is cheap hygiene but will not fix the
structural slowness. No spill-to-tempdb or `ColumnsWithNoStatistics` warnings were found.

---

## 7. Severity ranking

| Rank | Procedure | Primary root cause | Evidence | Worst symptom |
|---|---|---|---|---|
| 1 | **GetFilteredTickets** | `#Temp_Bookinginfo` built with no date filter + `CONVERT(DATE,col)` + `nvarchar(max)` AirlinePNRs convert | plan cost 24.4 (parallel temp insert); BookingInfo 6,640 reads; 702 ms statement; 15,094 avg reads | full-table tempdb load every call |
| 2 | **TicketList** | whole-`SegmentInfo` ROW_NUMBER + Key Lookup; 18 KB-row window-aggregate spool; AirlinePNRs/CreditNote converts | 323 ms avg CPU (parallel); plan-confirmed | wide-row spool + lookups scale badly |
| 3 | **AgentLedger** | `#UniqueBookingInfo` ranks all of `BookingInfo`; STRING_SPLIT estimate | plan #1 op cost 2.61; 53 execs × 164 ms; 3,767 reads | highest call frequency |
| 4 | **SP_ADMIN** | implicit-convert "Seek Plan" breakage; full SegmentInfo/Users scans; `SELECT *` temp load | plan-confirmed (24 ms / 1,940 reads); missing indexes 73.61 + 62.80 | structural scaling risk |
| — | `BTM.TicketList` | none observed — tables empty (0 rows) | plan max cost 0.0197 | plan not representative |

---

## 8. Data gaps — now CLOSED
1. ✅ Execution plans captured for `SP_ADMIN` and `dbo.TicketList`
   (`plan_SP_ADMIN_findings.md`, `plan_dbo_TicketList_findings.md`).
2. ✅ `dm_exec_procedure_stats` captured for all four procs in `stg_TakeoffPrelive`
   (`execution-stats-all-procs.txt`), plus cross-tenant context.
3. ✅ Statistics freshness captured (`stats-freshness.txt`) — §6.9.
4. ✅ Column datatypes captured (`column-types.txt`) — implicit conversions confirmed (§6.6).
5. ⏳ **Still open:** re-capture `BTM.TicketList` once `BTM.*` tables hold data (§5) — low
   priority (the dbo procedures are the live problem).
