# Performance Improvement — 10-Year-Old Enterprise Application

## Overview

A 10-year-old enterprise Java application had been going down every day for
17 consecutive days. It could not handle more than 5–10 concurrent users.
Simple report generation was timing out. Batch jobs that process file uploads
were running for 7 hours and causing DB deadlocks during business hours.

After investigation, multiple compounding failure points were identified.
Solutions were applied in waves — stopping the bleeding first, then fixing
the root causes, then modernizing for the long term.

**Result: 23+ incidents per month reduced to zero.**

```
Before                          After
──────────────────────────────  ──────────────────────────────
Concurrent users: 5-10          Concurrent users: 500+
Page load time:   2 minutes     Page load time:   < 2 seconds
Batch job runtime: 7 hours      Batch job runtime: < 15 minutes
Incidents/month:  23+           Incidents/month:  0
Storage (batch table): ~7 GB  Storage (batch table): ~700 MB
```

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Failure Points](#2-failure-points)
3. [Solution Waves](#3-solution-waves)
4. [Deep Dive — Each Fix](#4-deep-dive--each-fix)
5. [Architecture Evolution](#5-architecture-evolution)
6. [TimescaleDB — Database Layer](#6-timescaledb--database-layer)
7. [Results](#7-results)
8. [Technologies Used](#8-technologies-used)

---

## 1. Problem Statement

The application processes file uploads from external interfaces, inserts
records into a central database, and generates reports for business users.

```
External Interface
       │
       │  file upload (batch)
       ▼
  Java Application  ──────────────────►  PostgreSQL DB
       │                                  │
       │  report requests                 │  2.4 million rows
       ▼                                  │  growing 50K/month
  Business Users                          │  indexed, unpartitioned
  (timing out)                            │  8+ years of data
```

After 4+ years of data accumulation with no lifecycle management, the
database table had grown to 2.4 million rows. Every operation against
this table — inserts, queries, reports — was degrading. The application
had no ability to handle concurrent load and was collapsing daily.

---

## 2. Failure Points

Six distinct failure points were identified, each compounding the others.

---

### Failure Point 1 — Database Connections Never Closed

```
What was happening:
  Every DB operation opened a connection.
  The code never explicitly closed it.
  Connections accumulated until the DB server ran out.
  New requests hung waiting for a connection that never freed.

Impact:
  App hangs under any concurrent load.
  DB server connection exhaustion → full outage.
```

---

### Failure Point 2 — App and DB in Different Data Centers

```
What was happening:
  Application servers: Data Center A
  Database servers:    Data Center B
  Every SQL call crossed a WAN link.

  A query that takes 1ms locally took 80-120ms over WAN.
  A batch job doing 50,000 DB calls:
    Local:  50,000 × 1ms   =  50 seconds
    WAN:    50,000 × 100ms = 5,000 seconds (~1.4 hours)

Impact:
  Batch jobs running 7 hours partly due to pure network latency.
  Report queries taking minutes instead of seconds.
```

---

### Failure Point 3 — Connection Pool Size = 10

```
What was happening:
  HikariCP (or equivalent) pool configured with maxPoolSize = 10.
  Application deployed on servers handling hundreds of requests.
  10 concurrent DB operations max — everything else queued.

  Under 20 concurrent users:
    10 get DB connections → working
    10 queue waiting → timeout after N seconds → error
    Users see failures

Impact:
  Hard ceiling of ~10 concurrent users regardless of server capacity.
  Pool exhaustion triggered cascading timeouts and heap buildup.
```

---

### Failure Point 4 — Unoptimized Queries on a 2.4 Million Row Table

```
What was happening:
  The batch_data table had 2.4 million rows spanning 4 years.
  No partitioning. One monolithic table. One massive index.

  Index size at 2.4M rows: ~5 GB
  This index does not fit in the DB server's buffer pool.
  Every query causes repeated disk I/O to traverse the index.

  A typical report query:
    SELECT department, sum(amount)
    FROM batch_data
    WHERE created_date BETWEEN '2024-01-01' AND '2024-03-31'
    GROUP BY department;

  → Postgres opens the full 2.4M row table
  → Index thrashes the buffer pool
  → Query runs 20+ minutes
  → Holds DB connection for 20 minutes
  → Other queries queue behind it
  → App heap fills with waiting threads
  → Out of memory → crash

Impact:
  Single slow query could bring down the entire application.
  Cascading failure: one bad report = full outage.
```

---

### Failure Point 5 — Batch Insert on a Massive Indexed Table

```
What was happening:
  External interface uploads files.
  Batch job reads the file and inserts ~50,000 rows/month
  into the batch_data table (already 2.4M rows, indexed).

  Every INSERT must update the index.
  Index at 2.4M rows = ~5 GB, lives on disk.
  Each insert = random disk seek into the 5 GB index structure.
  50,000 inserts × disk seek = batch job runs 7 hours.

Impact:
  Batch job running 7 hours, overlapping with business hours.
  During overlap: DB locked for writes → deadlocks → app crashes.
  Batch never finished cleanly → data integrity risk.
```

---

### Failure Point 6 — Batch Jobs Scheduled During Business Hours

```
What was happening:
  Batch job scheduled at 11 PM PST.
  Job takes 7 hours to complete.
  Finishes at 6 AM PST.
  Business users start work at 8 AM PST.

  But the job regularly ran LONGER than 7 hours due to failures,
  retries, and DB contention. It frequently ran into 9-10 AM PST,
  directly overlapping with peak business usage.

  Batch writes + user reads on same unpartitioned table
  → DB deadlocks
  → Lock wait timeouts
  → Application errors for all users

Impact:
  DB deadlocks every morning.
  Batch and user traffic competing for the same locked resources.
```

---

## 3. Solution Waves

Solutions were applied in three waves, prioritized by severity.

```
WAVE 1 — Stop the Bleeding (Days 1-3)
  Stop the immediate outages. Fast fixes, no rework.
  ├── Close DB connections properly
  ├── Increase connection pool: 10 → 150
  └── Add 5-minute query timeout on SELECT queries

WAVE 2 — Fix Root Causes (Weeks 1-4)
  Address the underlying architectural problems.
  ├── Migrate app and DB to same data center
  ├── Redefine batch job schedule (11 PM → 4 PM trigger)
  ├── Implement retention policy (delete data > 8 years)
  └── Migrate batch_data to TimescaleDB hypertable

WAVE 3 — Modernize (Months 1-6)
  Prevent recurrence. Build for the next 10 years.
  ├── Parallel processing with CompletableFuture + virtual threads
  ├── Apache Kafka for async file processing
  ├── Modular monolith architecture
  ├── CI/CD pipeline with Jenkins
  └── Security scanning, unit testing, code quality gates
```

---

## 4. Deep Dive — Each Fix

### Fix 1 — Database Connection Closure

```java
// BEFORE — connection opened, never closed
Connection conn = dataSource.getConnection();
PreparedStatement ps = conn.prepareStatement(sql);
ResultSet rs = ps.executeQuery();
// process results...
// method returns — connection leaked forever

// AFTER — try-with-resources guarantees closure
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    // process results...
} // conn, ps, rs all closed automatically even on exception
```

**Impact:** Eliminated connection leak. DB server stopped running out of
connections under load.

---

### Fix 2 — Same Data Center Migration

```
Before:
  [App Server — DC West]  ──WAN──►  [DB Server — DC East]
  Latency: 80-120ms per call
  Batch job: 50K calls × 100ms = 5,000 seconds (~1.4 hours) of pure network wait

After:
  [App Server — DC East]  ──LAN──►  [DB Server — DC East]
  Latency: < 1ms per call
  Batch job: 50K calls × 1ms = 50 seconds of network time
```

**Impact:** Eliminated WAN latency as a compounding factor on every
DB operation.

---

### Fix 3 — Connection Pool Sizing

```yaml
# BEFORE
spring.datasource.hikari.maximum-pool-size: 10

# AFTER
spring.datasource.hikari.maximum-pool-size: 150
spring.datasource.hikari.minimum-idle: 20
spring.datasource.hikari.connection-timeout: 30000
spring.datasource.hikari.idle-timeout: 600000
```

**Impact:** Hard ceiling raised from 10 to 150 concurrent DB operations.
Concurrent user capacity increased immediately.

---

### Fix 4 — Query Timeout

```java
// BEFORE — no timeout, runaway queries consume resources indefinitely
PreparedStatement ps = conn.prepareStatement(sql);

// AFTER — 5 minute hard timeout on SELECT queries
PreparedStatement ps = conn.prepareStatement(sql);
ps.setQueryTimeout(300); // 300 seconds = 5 minutes
```

Also applied at DB level:

```sql
-- Postgres: kill any query running longer than 5 minutes
ALTER ROLE app_user SET statement_timeout = '5min';
```

**Impact:** Runaway 20-minute queries are killed at 5 minutes. Prevented
single slow queries from exhausting the connection pool and crashing the app.

---

### Fix 5 — Batch Job Rescheduling via Azure Functions

```
BEFORE:
  Scheduled cron at 11 PM PST
  Runs for 7 hours → finishes 6 AM PST on a good day
  On bad days → runs into 9-10 AM PST → deadlocks with users

AFTER:
  Azure Function triggered by file arrival event at 4 PM PST
  (end of business day)
  Runs during off-hours window: 4 PM → 4 AM (12 hour window)
  Completes well before 8 AM business start

Benefits:
  → No resource allocation when no file is present (serverless)
  → File-driven trigger replaces rigid time-based schedule
  → 7 additional hours of processing window
  → Zero overlap with business hours
```

---

### Fix 6 — Data Retention Policy

```
BEFORE:
  4+ years of data in batch_data with no lifecycle management.
  No retention rule defined. Data accumulated indefinitely.

AFTER:
  Worked with the business to define an 8-year retention rule.
  Migrated batch_data to a TimescaleDB hypertable (see Section 6).
  Applied an automated retention policy that runs nightly:

    SELECT add_retention_policy('batch_data',
        drop_after => INTERVAL '8 years'
    );

  TimescaleDB drops entire monthly chunk files instantly.
  No row-by-row DELETE. No WAL bloat. No table locking.
  No DBA involvement. Cleaned 60% of data volume on first run.
  Self-managing from that point forward.
```

---

### Fix 7 — Parallel Processing

```java
// BEFORE — sequential processing of file records
for (Record record : records) {
    processRecord(record);   // one at a time, blocking
}

// AFTER — parallel processing with CompletableFuture
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

List<CompletableFuture<Void>> futures = records.stream()
    .map(record -> CompletableFuture.runAsync(
        () -> processRecord(record), executor))
    .collect(Collectors.toList());

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
```

**Impact:** Batch processing time reduced significantly. Java 21 virtual
threads (Project Loom) allowed high concurrency without thread pool
exhaustion.

---

### Fix 8 — Apache Kafka for Async File Processing

```
BEFORE:
  File upload → synchronous processing → HTTP response blocked
  User waits for entire file to process before getting response
  Large files: request times out → user retries → duplicate processing

AFTER:
  File upload → Kafka topic message → immediate HTTP 202 Accepted
  Consumer group processes file asynchronously
  User gets callback/notification when complete

  [File Upload API] ──► [Kafka: file-upload-events] ──► [Batch Consumer]
                                                              │
                                                              ▼
                                                        [batch_data table]
```

**Impact:** Upload endpoint no longer blocks. File processing failures
are retried by Kafka without user involvement. Heavy processing decoupled
from the web tier.

---

### Fix 9 — Modular Monolith Migration

```
BEFORE — Big Ball of Mud monolith:
  All business logic, DB access, file processing, reporting
  in one tangled codebase. A failure in any module
  brought down the entire application.

AFTER — Modular Monolith:
  ┌─────────────────────────────────────────┐
  │           Application                   │
  │  ┌──────────┐  ┌──────────┐  ┌───────┐ │
  │  │  Batch   │  │ Reports  │  │ Users │ │
  │  │  Module  │  │  Module  │  │ Module│ │
  │  └──────────┘  └──────────┘  └───────┘ │
  │       │              │            │     │
  │       └──────────────┴────────────┘     │
  │                  Shared DB              │
  └─────────────────────────────────────────┘

  Each module has defined boundaries.
  A failure in Batch Module does not crash Reports Module.
  Enables independent deployability in future microservices migration.
```

---

## 5. Architecture Evolution

```
BEFORE — Single server, single table, everything coupled:

  Users ──► [Java App — DC West] ──WAN──► [PostgreSQL — DC East]
                                               │
                                          batch_data
                                          2.4M rows
                                          no partitioning
                                          no lifecycle mgmt


AFTER — Decoupled, event-driven, same data center:

  Users ──► [Java App — DC East] ──LAN──► [PostgreSQL — DC East]
                                               │
  Files ──► [Azure Function]                   │  batch_data
               │                               │  hypertable
               ▼                               │  chunked by month
           [Kafka Topic] ──► [Consumer] ───────┘  compressed > 3 months
                                                  auto-retained 8 years
```

---

## 6. TimescaleDB — Database Layer

### Why TimescaleDB Was Applied to This Table

```
batch_data table characteristics:
  ✓ Every row has a timestamp (created_date)
  ✓ Rows only ever inserted — never updated
  ✓ 2.4 million rows, growing 50,000/month, forever
  ✓ Reports always filter by date range
  ✓ Old data (8+ years) should be deleted automatically
  ✓ Recent data queried most, old data rarely touched

These characteristics made TimescaleDB the right choice for this table.
```

### The Core Problem at the Database Level

```
2.4 million rows, one table, one index:

  ┌──────────────────────────────────────────────────────┐
  │  batch_data  (vanilla Postgres)                      │
  │                                                      │
  │  2,400,000 rows  ──►  index size: ~5 GB           │
  │                                                      │
  │  INSERT:  must update 5 GB index → disk seek        │
  │  SELECT:  index too large for buffer pool → thrash   │
  │  DELETE:  row-by-row scan of 2.4M rows → hours       │
  └──────────────────────────────────────────────────────┘
```

### What TimescaleDB Does Instead

```
Same 2.4 million rows, split into monthly chunks:

  Jan 2015    Feb 2015    Mar 2015  ...  Mar 2025   Apr 2025
  ┌────────┐  ┌────────┐  ┌────────┐    ┌────────┐  ┌────────┐
  │chunk_1 │  │chunk_2 │  │chunk_3 │    │chunk_N │  │active  │
  │~200K rows│  │~200K rows│  │~200K rows│    │~200K rows│  │growing │
  │COMPRESS│  │COMPRESS│  │COMPRESS│    │COMPRESS│  │  raw   │
  │ ~50 MB │  │ ~50 MB │  │ ~50 MB │    │ ~50 MB │  │        │
  └────────┘  └────────┘  └────────┘    └────────┘  └────────┘
       ↑
  Fits in RAM entirely.
  INSERT only touches active chunk index.
  Report query skips 45 months, reads only the 3 needed.
```

### Migration Applied

```sql
-- Step 1: Enable the extension (once per database)
CREATE EXTENSION IF NOT EXISTS timescaledb;


-- Step 2: Convert to hypertable
-- migrate_data => TRUE migrates existing 2.4M rows automatically
SELECT create_hypertable('batch_data', 'created_date',
    chunk_time_interval => INTERVAL '1 month',
    migrate_data        => TRUE
);
-- Executed during maintenance window — one-time migration


-- Step 3: Configure compression
-- segmentby = 'department' — reports filter by department
--   → each department gets its own compressed block
--   → report for Dept A never touches Dept B data
-- orderby = 'created_date DESC' — recent rows at top of each block
--   → queries for recent data stop scanning early
ALTER TABLE batch_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'department',
    timescaledb.compress_orderby   = 'created_date DESC'
);

-- Compress chunks older than 3 months
-- Recent 3 months stay raw — fast inserts for the active batch job
SELECT add_compression_policy('batch_data',
    compress_after => INTERVAL '3 months'
);


-- Step 4: Retention policy — replaces manual DELETE forever
SELECT add_retention_policy('batch_data',
    drop_after => INTERVAL '8 years'
);
-- Runs automatically every night. No DBA involvement. No DELETE storms.


-- Step 5: Continuous aggregate for reports
-- Pre-computes monthly summaries in the background.
-- Report queries hit this view: ~48 rows instead of 2.4M.
CREATE MATERIALIZED VIEW monthly_batch_summary
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', created_date) AS month,
    department,
    sum(amount)              AS total_amount,
    count(*)                 AS record_count,
    count(DISTINCT batch_id) AS batch_count
FROM batch_data
GROUP BY month, department;

SELECT add_continuous_aggregate_policy('monthly_batch_summary',
    start_offset      => INTERVAL '3 months',
    end_offset        => INTERVAL '1 month',
    schedule_interval => INTERVAL '1 day'
);
```

### Problems Solved at the DB Layer

```
PROBLEM: Batch INSERT takes 7 hours
  Vanilla Postgres: INSERT updates 5 GB index → disk seeks
  TimescaleDB:      INSERT updates active chunk index (~50 MB, in RAM)
  Result:           Batch job from 7 hours → under 15 minutes

PROBLEM: Report queries take 20 minutes
  Vanilla Postgres: Full scan of 2.4M rows → buffer pool thrash
  TimescaleDB:      Chunk exclusion skips 45 of 48 months
                    Continuous aggregate pre-computes summaries
  Result:           Report query from 20 minutes → < 2 seconds

PROBLEM: Manual DELETE of 8-year-old data was painful
  Vanilla Postgres: Row-by-row DELETE, WAL bloat, hours of runtime
  TimescaleDB:      Entire chunk files dropped instantly, zero scan
                    Automated nightly — zero DBA effort forever
  Result:           Retention managed automatically, zero incidents

PROBLEM: Storage at 2.4M rows was ~7 GB
  Vanilla Postgres: No compression, full row storage
  TimescaleDB:      Columnar compression on chunks > 3 months old
                    Similar values (prices, amounts) compress 90%
  Result:           ~7 GB → ~700 MB
```

### The Three Dials Configured for This App

```
DIAL 1: chunk_time_interval = '1 month'
  50,000 rows inserted per month → ~200K rows per chunk after 4 years
  Monthly chunks align with report query patterns (monthly reports)
  Each chunk index fits in RAM: ~50 MB

DIAL 2: compress_after = '3 months'
  Keep last 3 months raw → active batch inserts are fast
  Anything older than 3 months → compress to columnar format
  Reports on old data hit compressed columnar blocks (faster reads)

DIAL 3: background job schedule = '1 day' (default)
  Compression worker checks nightly
  Retention worker checks nightly
  No manual intervention needed
```

### Storage Lifecycle

```
Month 0-3:   Raw chunk — fast inserts, raw row format
             ┌──────────────────────────┐
             │  row: [date][dept][amt]  │  × 200K rows = ~120 MB/chunk
             └──────────────────────────┘

Month 3+:    Compressed chunk — columnar format, 90% smaller
             ┌──────────────────────────┐
             │  dates:   [block]        │
             │  depts:   [block]        │  ~12 MB/chunk
             │  amounts: [block]        │
             └──────────────────────────┘

Year 8+:     Dropped instantly
             retention job drops entire chunk file
             no row scanning, no WAL, no vacuum
```

---

## 7. Results

### Quantitative Improvements

| Metric | Before | After |
|---|---|---|
| Incidents per month | 23+ | 0 |
| Concurrent users | 5-10 | 500+ |
| Page load time | ~2 minutes | < 2 seconds |
| Batch job runtime | 7 hours | < 15 minutes |
| Report query time | 20+ minutes | < 2 seconds |
| DB storage (batch table) | ~7 GB | ~700 MB |
| Connection pool | 10 | 150 |
| Data retention | Manual, ad hoc | Automated, nightly |
| Processing improvement | baseline | 10-20% increase |
| Execution time reduction | baseline | 20-30% reduction |

### What Each Wave Delivered

```
WAVE 1 — Immediate (Days 1-3):
  Incidents stopped within 72 hours.
  App stable enough for business to operate.
  Fixes: connection closure, pool size, query timeout.

WAVE 2 — Structural (Weeks 1-4):
  Root causes eliminated.
  Batch job no longer conflicts with business hours.
  60% data volume reduction from retention policy.
  Fixes: same data center, batch rescheduling, retention.

WAVE 3 — Long Term (Months 1-6):
  Performance ceiling raised by order of magnitude.
  App modernized for next 10 years.
  Fixes: parallel processing, Kafka, modular architecture,
         CI/CD, TimescaleDB migration.
```

---

## 8. Technologies Used

| Category | Technology |
|---|---|
| Language | Java 21 |
| Async Processing | Java 8 CompletableFuture, Virtual Threads (Project Loom) |
| Messaging | Apache Kafka |
| Database | PostgreSQL + TimescaleDB extension |
| Scheduling | Azure Functions (file-triggered) |
| Connection Pool | HikariCP |
| Code Quality | SonarQube |
| Security Scanning | Nexus IQ, Whitehat DAST, SAST |
| Build | Maven, Artifactory |
| CI/CD | Jenkins |
| Testing | JUnit |
| Version Control | Git |
| Diagnostics | Heap Dump Analyzer, SQL Query Analyzer |
