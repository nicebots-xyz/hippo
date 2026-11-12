## hippo@0.2.1

### Upgrade PostgreSQL to 18.7 ([#55](https://github.com/nicebots-xyz/hippo/pull/55))

**Security.** 18.7 fixes 52 CVEs ([announcement](https://www.postgresql.org/about/news/postgresql-187-1712-1616-1520-1425-and-19-rc-2-released-3442/)). The most severe let a role that can create objects run code as the operating system user running the server: SQL-language function output type confusion (CVE-2026-18447), statistics import type confusion (CVE-2026-116329, 18 only) and dropped-column type confusion (CVE-2026-120142). Others let dblink and subscriptions append to arbitrary files (CVE-2026-14661) and bypass row security through `INSERT ... ON CONFLICT` (CVE-2026-120154). Redeploy with the new image.

No dump or restore is needed, but 18.7 fixes several index bugs whose damage it does not repair. After upgrading, reindex `contrib/pg_trgm` `gist_trgm_ops` indexes and B-tree indexes on unconstrained `bpchar` columns, and reindex any GIN index you suspect is affected (CVE-2026-116336). See the [18.7 release notes](https://www.postgresql.org/docs/18/release-18-7.html).

### Package changes

<details>
<summary>12 packages changed in the image</summary>

- `apt` (deb): 3.0.3 → 3.0.4
- `libapparmor1` (deb): 4.1.0-1 (added)
- `libapt-pkg7.0` (deb): 3.0.3 → 3.0.4
- `libpq5` (deb): 18.6-1.pgdg13+2 → 18.7-1.pgdg13+2
- `libsystemd-shared` (deb): 257.13-1~deb13u1 (added)
- `postgresql-18` (deb): 18.6-1.pgdg13+2 → 18.7-1.pgdg13+2
- `postgresql-18-jit` (deb): 18.6-1.pgdg13+2 → 18.7-1.pgdg13+2
- `postgresql-client-18` (deb): 18.6-1.pgdg13+2 → 18.7-1.pgdg13+2
- `postgresql-client-common` (deb): 293.pgdg13+1 → 294.pgdg13+2
- `postgresql-common` (deb): 293.pgdg13+1 → 294.pgdg13+2
- `systemd` (deb): 257.13-1~deb13u1 (added)
- `tzdata` (deb): 2026e-0+deb13u1 → 2026f-0+deb13u1

</details>

## hippo@0.2.0

### Members are named after their node, not their task slot ([#53](https://github.com/nicebots-xyz/hippo/pull/53))

`deploy/stack.yml` named members and their volumes after the Swarm task slot (`pg-1`, `pgdata-1`). Swarm can move a slot to another node but a local volume stays behind, so a member could restart on an empty volume or on a stale copy left by an earlier placement, and stale etcd members could roll the cluster state back. Members are now `pg-<host>` and `etcd-<host>`, each node keeps one `hippo_pgdata` and one `hippo_etcd-data` volume, and the stack needs `HIPPO_NODE_1`, `HIPPO_NODE_2` and `HIPPO_NODE_3` set to the hostnames of the `hippo.pg=true` nodes. The stack's config key is now `patroni_v3`.

**Breaking.** To migrate a running cluster:

1. Note which `pgdata-<slot>` each node's running `pg` task uses: `docker service ps hippo_pg` lists `hippo_pg.<slot>` per node.
2. `docker stack rm hippo` and wait for the tasks to stop.
3. On each node, copy that volume: `docker run --rm -u 0 -v pgdata-<slot>:/from -v hippo_pgdata:/to "$HIPPO_IMAGE" cp -a /from/. /to/`. etcd starts empty, since its member names change, and Patroni rebuilds its state in etcd from the data directories.
4. Deploy with the new variables. Once `patronictl list` shows a leader and two streaming replicas, remove the old `pgdata-*` and `etcd-data-*` volumes on every node.

## hippo@0.1.2

### Update lock file images/base/requirements.lock ([#45](https://github.com/nicebots-xyz/hippo/pull/45))



### Upgrade dependency psycopg to v3.3.7 ([#47](https://github.com/nicebots-xyz/hippo/pull/47))



### Package changes

<details>
<summary>8 packages changed in the image</summary>

- `certifi` (pip): 2026.10.19 → 2026.10.23
- `dnspython` (pip): 2.9.0 → 2.9.1
- `psycopg` (pip): 3.3.6 → 3.3.7
- `py-consul` (pip): 1.7.1 → 1.8.0
- `python-dateutil` (pip): 2.9.0.post0 → 4.0.0
- `six` (pip): 1.17.0 (removed)
- `tzdata` (pip): 2026.6 (added)
- `wcwidth` (pip): 0.9.3 → 0.9.4

</details>

## hippo@0.1.1

### Passwords ignore a trailing newline in the secrets ([#27](https://github.com/nicebots-xyz/hippo/pull/27))

`deploy/patroni.yml.tmpl` reads the superuser and replication passwords as folded scalars, so a secret created with a trailing newline (`openssl rand -hex 32 | docker secret create …`) no longer becomes a password ending in a space that clients reading the secret with `$(cat …)` cannot match. Secrets without a newline give the same password as before. The stack's config key is now `patroni_v2`.

Clusters bootstrapped from such a secret keep the old password: run `ALTER ROLE` for `postgres` and `replicator` on the leader before redeploying.

### Upgrade docker/dockerfile docker tag to v1.28.0 ([#11](https://github.com/nicebots-xyz/hippo/pull/11))



### Upgrade dependency etcd-io/etcd to v3.7.3 ([#6](https://github.com/nicebots-xyz/hippo/pull/6))



### Upgrade haproxy docker tag to v3.4.7 ([#12](https://github.com/nicebots-xyz/hippo/pull/12))

## hippo@0.1.0

### Apps connect on their own network ([#24](https://github.com/nicebots-xyz/hippo/pull/24))

Apps join the `hippo-clients` network and reach Postgres through HAProxy (`haproxy:5000` for the primary, `:5001` for the replicas). The cluster network `hippo` is no longer attachable, so apps can't reach etcd or Patroni's REST API, and Patroni only accepts state-changing REST calls (restart, failover, configuration, …) from cluster members.

Both networks are encrypted: traffic between nodes goes over IPsec, so IP protocol 50 (ESP) must be allowed between them.

### Images link to their source repository ([#13](https://github.com/nicebots-xyz/hippo/pull/13))

Both images carry the `org.opencontainers.image.source` label, so registries link them to `nicebots-xyz/hippo`.

### Initial release ([#1](https://github.com/nicebots-xyz/hippo/pull/1))

HA PostgreSQL 18 for Docker Swarm: Patroni 4.1.5 + etcd + HAProxy, deployed with a single `docker stack deploy`.

- `base` image: PostgreSQL 18 with its bundled extensions and Patroni.
- `ext` image: `base` plus pgvector, pg_cron, pg_partman, pgaudit, pg_repack, hypopg and pg_stat_kcache.
- Reference stack in `deploy/`, validated by a smoke test that rolls every Postgres member under write load without losing an acknowledged write.
- MIT licensed, with an SPDX header on every source file.

### Pin the Dockerfile frontend to an exact version ([#10](https://github.com/nicebots-xyz/hippo/pull/10))

Images are built with `docker/dockerfile:1.27.1` by digest instead of the moving `1` tag, so frontend updates arrive as versioned, cooled-down updates.

### Patroni uses the system libpq ([#16](https://github.com/nicebots-xyz/hippo/pull/16))

Patroni's driver is the pure-Python `psycopg` on Debian's `libpq5` instead of the `psycopg-binary` wheel, which bundled its own copies of libpq, OpenSSL, Kerberos and LDAP built for CentOS 7, without license metadata.

### Lock Patroni's Python dependencies with a 7-day cooldown ([#3](https://github.com/nicebots-xyz/hippo/pull/3))

The lock only takes releases at least 7 days old, matching Renovate's cooldown for PyPI updates.
Renovate keeps the lock up to date through lock file maintenance.

### The Python lock file no longer stays in an image layer ([#21](https://github.com/nicebots-xyz/hippo/pull/21))

The base image mounts `requirements.lock` for the Patroni install instead of copying it in, so the file is no longer kept in a layer of its own.

### SBOM covering everything in the images ([#17](https://github.com/nicebots-xyz/hippo/pull/17))

Images are published with SPDX SBOM and build provenance attestations. The SBOM lists every package and every component no package manager owns (gosu, the postgres entrypoint scripts, hippo's own tooling), each with its license, and releases are refused when anything in the image is missing from it. See `docs/licenses.md`.
