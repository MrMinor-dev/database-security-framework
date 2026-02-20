# Database Security Framework

**Production-tested database security architecture for autonomous AI agents — Row-Level Security, statement-type whitelisting, column-level write permissions, and least-privilege enforcement across 50+ tables.**

---

## The Problem

When an AI agent needs database access, the question isn't *whether* to grant it — it's *which tables, which columns, which operations, with what constraints*.

Most AI-database integrations fall into two camps: full access (dangerous) or no access (useless). Neither works in production. An autonomous agent that manages financial workflows, compliance checks, and content operations needs to read broadly but write surgically. It needs SELECT access across dozens of tables but INSERT permission on exactly four. It needs DELETE capability on three specific tables — and only with a WHERE clause.

This framework solves that problem across a 50+ table PostgreSQL schema that evolved through 14 versions without a single data loss event. It implements defense-in-depth at the database layer: Row-Level Security on every table, statement-type blocking on every query, column-level whitelisting on every write, and output limits on every result set.

The hardest part wasn't the technical implementation. It was discovering what was missing. A security audit surfaced 8 actual errors hiding in production — 5 tables without RLS, 3 views with SECURITY DEFINER exposure — in a system that appeared secure from every other angle.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                 Agent Request                     │
└──────────────────────┬──────────────────────────┘
                       │
        ┌──────────────▼──────────────┐
        │    Input Router (Layer 0)    │
        │  MCP vs Webhook detection    │
        │  Defensive type-checking     │
        └──────────────┬──────────────┘
                       │
          ┌────────────▼────────────┐
          │  Operation Classifier    │
          │  READ path / WRITE path  │
          └─────┬──────────┬────────┘
                │          │
    ┌───────────▼───┐  ┌───▼───────────┐
    │  Safe SQL      │  │  Safe DB       │
    │  Query         │  │  Write         │
    │                │  │                │
    │  • SELECT only │  │  • Table       │
    │  • 7 blocked   │  │    whitelist   │
    │    statements  │  │  • Column      │
    │  • 50-row      │  │    whitelist   │
    │    output cap  │  │  • WHERE req   │
    │                │  │    on DELETE   │
    └───────┬───────┘  └───────┬───────┘
            │                  │
    ┌───────▼──────────────────▼───────┐
    │     PostgreSQL + Row-Level        │
    │         Security (RLS)            │
    │                                   │
    │  • RLS on all production tables   │
    │  • Service role policies          │
    │  • 4 namespaces (haios_, aos_,    │
    │    tgt_, mi_)                     │
    │  • Audit columns on all tables    │
    └───────────────────────────────────┘
```

### Components

**Input Router (Layer 0)** — Framework-level defensive routing that detects whether a request arrives via MCP `execute_workflow` or webhook `curl`, which inject payloads through different structures (`chatInput` vs `body.query`). Type-checks both formats before any database operation executes. This bug was invisible at the application layer — only diagnosable by tracing invocation paths across framework boundaries.

**Safe SQL Query (Read Path)** — Webhook-triggered read-only service. Blocks 7 statement types (INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE, CREATE). Enforces 50-row output limits. Returns structured results with row counts and metadata.

**Safe DB Write (Write Path)** — Webhook-triggered write service with per-table permissions:
- **INSERT whitelist:** 4 specific tables with defined allowed columns
- **UPDATE whitelist:** 6 tables with column-level restrictions
- **DELETE:** Restricted to 3 tables, mandatory WHERE clause requirement
- Every write validated against whitelist before execution

**Row-Level Security** — RLS enabled on all production tables. Service role policies for controlled agent access. Security audit methodology that systematically checks every table and view for policy gaps.

**Schema Governance** — 14 additive-only migrations (v1 → v7.14) with mandatory 7-step change checklist:
1. Document the change in schema master
2. Verify backward compatibility
3. Write migration SQL (additive only)
4. Test in isolation
5. Execute migration
6. Verify data integrity
7. Update dependent documents

---

## Key Insight

**Security audits find what architecture diagrams miss.**

The system looked secure from every design document. RLS was "enabled." Access controls were "in place." But a systematic audit — checking every table, every view, every policy — surfaced 8 real errors:

- 5 tables missing Row-Level Security entirely (`api_costs`, `system_flags`, `compliance_checks`, `prohibited_products`, `health_checks`)
- 3 views using SECURITY DEFINER (executing with creator privileges instead of caller privileges)

These weren't theoretical vulnerabilities. They were production gaps in a system that had been operating for months. The fix took minutes. Finding them required a methodology that questions what "enabled" actually means at the individual table level.

The same pattern appeared at the framework layer. Intermittent query failures traced not to SQL or application logic, but to the invisible difference between how MCP and webhook protocols inject request bodies. The bug was invisible at every layer except the one where two frameworks meet.

**Lesson:** "Secured" is not a binary state. It's a claim that requires table-by-table, view-by-view, invocation-path-by-invocation-path verification.

---

## Results

| Metric | Value |
|--------|-------|
| Schema versions | 14 (v1 → v7.14) |
| Production tables | 50+ |
| Data loss events | 0 |
| Namespaces | 4 (`haios_`, `aos_`, `tgt_`, `mi_`) |
| Blocked statement types | 7 |
| Tables with RLS | All production tables |
| Security errors found via audit | 8 (5 missing RLS + 3 SECURITY DEFINER) |
| Write-whitelisted tables | 6 (column-level) |
| INSERT-allowed tables | 4 (specific columns) |
| DELETE-restricted tables | 3 (WHERE clause required) |
| Query output limit | 50 rows |
| Utility workflows standardized | 3 (Safe SQL, Semantic Search, Safe DB Write) |

---

## The Framework-Level Bug

Worth calling out specifically: the input routing problem.

n8n's MCP `execute_workflow` and webhook `curl` inject request bodies through different payload structures. MCP sends `chatInput` at one path. Webhooks send `body.query` at another. Both arrive at the same workflow node.

The result: intermittent database query failures that appeared random. Sometimes queries worked, sometimes they returned empty results. The application logic was correct. The SQL was correct. The failure only appeared when the *invocation method changed* — something invisible to every layer except the boundary between two frameworks.

The fix: defensive input routing that type-checks both formats.

```
$input.first().json.body?.query || $input.first().json.chatInput || ''
```

**This pattern was standardized across all 3 utility workflows** — because if it happened once, it would happen everywhere the same two systems meet.

---

## Why It Matters

Least-privilege access control for autonomous agents is the same problem as least-privilege access control for any infrastructure. The questions are identical: which identities get access to which resources, at which operation level, with what conditions attached. The answers here happen to be about an AI agent and a PostgreSQL schema — but the pattern (allowlist by table, restrict by column, require WHERE clauses on destructive operations) is the standard enterprise security model applied to a new kind of principal.

The audit methodology is the transferable part. "RLS is enabled" is a design claim. Whether it's actually enabled on every table, every view, across every invocation path — that's a verification question. The 8 errors found weren't in the design. They were in the gap between what the architecture said and what the production system did. That gap is where every real security failure lives. The methodology for finding it — systematic, table-by-table, checking actual state rather than documented intent — applies to any system where "secured" is a claim rather than a measurement.

---

## Built With

- **Database:** Supabase (PostgreSQL + pgvector)
- **Security:** Row-Level Security, service role policies
- **Automation:** n8n (webhook-triggered services)
- **Protocol:** Model Context Protocol (MCP)
- **Sessions:** ~50 sessions of iterative schema evolution and security hardening

Part of [HAIOS](https://mrminor-dev.github.io) — a Human-AI Operating System in production since October 2024.

---

## License

MIT

## Author

**Jordan Waxman** — [mrminor-dev.github.io](https://mrminor-dev.github.io)

14 years operations leadership — building human-AI infrastructure since 2025. The intersection is the moat.
