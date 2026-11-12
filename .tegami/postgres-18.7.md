---
packages:
  hippo: patch
---

### Upgrade PostgreSQL to 18.7

**Security.** 18.7 fixes 52 CVEs ([announcement](https://www.postgresql.org/about/news/postgresql-187-1712-1616-1520-1425-and-19-rc-2-released-3442/)). The most severe let a role that can create objects run code as the operating system user running the server: SQL-language function output type confusion (CVE-2026-18447), statistics import type confusion (CVE-2026-116329, 18 only) and dropped-column type confusion (CVE-2026-120142). Others let dblink and subscriptions append to arbitrary files (CVE-2026-14661) and bypass row security through `INSERT ... ON CONFLICT` (CVE-2026-120154). Redeploy with the new image.

No dump or restore is needed, but 18.7 fixes several index bugs whose damage it does not repair. After upgrading, reindex `contrib/pg_trgm` `gist_trgm_ops` indexes and B-tree indexes on unconstrained `bpchar` columns, and reindex any GIN index you suspect is affected (CVE-2026-116336). See the [18.7 release notes](https://www.postgresql.org/docs/18/release-18-7.html).
