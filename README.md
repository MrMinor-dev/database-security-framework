# Database Security — RLS, Allowlisting, Least Privilege

I ran a security audit on a system I'd been building for months and found 8 actual errors.

Five tables had no Row-Level Security. Three views were running with creator privileges instead of caller privileges — meaning any query on those views executed with the same access level as whoever created them, not whoever was calling. The system looked secure from every design document. It wasn't.

That gap between documented intent and production reality is where every real security failure lives.

I built two services that control exactly what the AI can touch. Read queries block seven statement types at the string-parsing layer — if the query text contains DELETE, it never reaches the database, regardless of how the request was framed. Writes go through a per-table, per-column allowlist. The AI can update content fields but can't touch `id`, `created_at`, or `compliance_status`. The allowlist is the access boundary, not a policy someone agreed to follow.

One bug took the longest to find: intermittent query failures with no obvious pattern. Traced not to SQL or application logic, but to the difference between how MCP and webhook protocols inject request bodies. Same workflow, two different call paths, two different payload shapes. The fix is one line of defensive type-checking, now in every utility workflow.

**The methodology scales to any system where AI agents need surgical access.** Broad enough to run the business, narrow enough to be safe.

## What's Here

- Read query service with 7-statement type blocking
- Write service with per-table, per-column allowlist
- Row-Level Security enforcement on all tables
- Security audit methodology (8 errors found and fixed)
- Framework-boundary input routing fix

## Built With

Supabase (PostgreSQL + RLS) · n8n · Claude (Anthropic) · Model Context Protocol (MCP)

## Author

Jordan Waxman — [jordanwaxman.com](https://jordanwaxman.com) · [LinkedIn](https://www.linkedin.com/in/waxmanjordan/)
