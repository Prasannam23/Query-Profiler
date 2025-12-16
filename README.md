# Query Profiler — PostgreSQL Query Impact Analyzer

A **production-grade PostgreSQL query profiling tool** that identifies **high-impact SQL queries** by combining **execution frequency** and **latency** using accurate, time-based metrics.

> **Core Question Answered:**  
> *Which queries are hurting my database the most right now?*

---

## Why This Tool Exists

Most teams struggle with database performance not because queries are slow **once**, but because:
- They run **too frequently**
- They consume **significant cumulative DB time**
- Traditional logs do not show *impact over time*

`Query Profiler` uses **PostgreSQL internal statistics** to compute **real, time-windowed impact scores**, making it easier to:
- Identify bottlenecks
- Prioritize optimizations
- Avoid misleading one-off slow queries

---

## Key Features

- Accurate **calls-per-minute** calculation using delta snapshots
- Impact scoring based on **frequency × latency**
- Safe **read-only** PostgreSQL access
- CLI-first design (scriptable, automatable)
- Production-ready architecture (config, logging, retries)

---

## How It Works (High Level)

```
PostgreSQL
   |
   |  pg_stat_statements (cumulative stats)
   |
Collector  →  Snapshot Engine  →  Analyzer  →  Output
```

1. **Collector** fetches raw query statistics  
2. **Snapshot Engine** computes deltas over time  
3. **Analyzer** calculates impact score & severity  
4. **Exporter** displays results (CLI / JSON)

---

## Accurate Metrics (Important)

PostgreSQL’s `pg_stat_statements` stores **cumulative counters**, not rates.

This tool:
- Takes **periodic snapshots**
- Computes **deltas between snapshots**
- Divides by **real elapsed time**
- Detects counter resets safely

This ensures:
- Calls/min are real
- Latency is not misleading
- High-impact queries are correctly identified

---

## Impact Score Logic

### Formula
```
impact_score = calls_per_minute × avg_latency_ms
```

This answers:
> “How much DB time does this query consume per minute?”

### Severity Levels

| Impact Score | Level |
|-------------|-------|
| > 1,000,000 | CRITICAL |
| > 300,000   | HIGH |
| > 100,000   | MEDIUM |
| else        | LOW |

---

## Example Output

```
Query Hash: 987654321
Calls/min: 8,200
Avg Latency: 96ms
Impact Score: HIGH
-------------------------------------------------
SELECT * FROM orders WHERE user_id = $1
```

---

## Repository Structure

```
query-profiler/
├── cmd/
│   └── qp/
│       └── main.go            # CLI entry point
│
├── internal/
│   ├── config/
│   │   └── config.go          # Config loading (env, yaml, flags)
│   │
│   ├── db/
│   │   └── postgres.go        # PostgreSQL connection (read-only)
│   │
│   ├── collector/
│   │   └── pgstat.go          # pg_stat_statements collector
│   │
│   ├── snapshot/
│   │   └── engine.go          # Snapshot + delta computation
│   │
│   ├── analyzer/
│   │   ├── impact.go          # Impact score calculation
│   │   └── rank.go            # Sorting & severity classification
│   │
│   ├── model/
│   │   └── query.go           # Core data structures
│   │
│   ├── exporter/
│   │   ├── cli.go             # CLI table output
│   │   └── json.go            # JSON output
│   │
│   └── logger/
│       └── logger.go          # Structured logging
│
├── configs/
│   └── config.yaml            # Default configuration
│
├── scripts/
│   │   └── enable_pg_stat.sql # pg_stat_statements setup
│
├── Dockerfile
├── Makefile
├── go.mod
└── README.md
```

---

## Installation

### Enable pg_stat_statements

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

Ensure PostgreSQL config:
```
shared_preload_libraries = 'pg_stat_statements'
```

Restart PostgreSQL.

---

### Create Read-Only User (Recommended)

```sql
CREATE ROLE qp_reader WITH LOGIN PASSWORD 'strong_password';
GRANT CONNECT ON DATABASE appdb TO qp_reader;
GRANT USAGE ON SCHEMA public TO qp_reader;
GRANT SELECT ON pg_stat_statements TO qp_reader;
```

---

## Configuration

```yaml
db:
  host: localhost
  port: 5432
  name: appdb
  user: qp_reader
  password: secret

poll_interval: 30s
top_n: 10
```

Config priority:
1. ENV
2. YAML
3. CLI flags

---

## Usage

```bash
qp analyze --config=configs/config.yaml
```

JSON output:
```bash
qp analyze --json
```

---

## Safety & Production Guarantees

- Read-only DB access
- Snapshot-based rate calculation
- Reset detection
- Graceful failure handling
- Structured logs
- Low memory footprint

---

## Future Enhancements

- HTTP API
- Prometheus metrics
- p95 / p99 latency
- Alert thresholds
- Historical persistence
- Multi-database support

---

## License

[MIT](./LICENSE)
