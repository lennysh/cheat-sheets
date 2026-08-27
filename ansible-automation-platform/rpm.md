# AAP — RPM

Traditional installer on RHEL (`setup.sh`). Control-plane processes run as systemd / supervisord; `awx-manage` is on the PATH as the `awx` user.

Sibling sheets: [containerized](containerized.md) · [OpenShift](openshift.md) · [shared API](README.md)

## Contents

1. [Status](#1-status)
2. [Logs](#2-logs)
3. [Database](#3-database)
4. [Migrations](#4-migrations)
5. [Redis](#5-redis)
6. [Receptor / mesh](#6-receptor--mesh)
7. [Configuration](#7-configuration)
8. [Diagnostic bundle](#8-diagnostic-bundle)
9. [Common scenarios](#9-common-scenarios)

---

## 1. Status

**When:** UI down, jobs not starting, after reboot or installer rerun.

```bash
automation-controller-service status
supervisorctl status
systemctl status --no-pager automation-controller automation-gateway redis receptor postgresql

# What is actually listening
ss -tulpn | grep -E '443|80|5432|6379|27199'
ps aux | grep -E 'awx|ansible|receptor|redis|postgres' | grep -v grep
```

**Mutating** — bounce the controller stack:

```bash
automation-controller-service restart
# or a single supervisord program:
# supervisorctl restart <program>
```

---

## 2. Logs

**When:** Need recent errors, dispatcher/callback noise, or proof a migration is running.

```bash
tail -f /var/log/tower/tower.log
grep -i error /var/log/tower/tower.log | tail -50
grep -i migrating /var/log/tower/tower.log | tail -20

journalctl -u automation-controller -u automation-gateway -n 200 --no-pager
journalctl -p err -n 50 --no-pager
```

Dispatcher / callback receiver are mixed into `tower.log` on RPM; grep if you need to split them.

---

## 3. Database

**When:** Controller will not start, jobs fail with DB errors, or you need size / connection counts.

Run as `awx` so `awx-manage` picks up `/etc/tower` settings. External Postgres: use `psql` with the host from `postgres.py` instead.

```bash
sudo -u awx awx-manage dbshell -c "SELECT version();"

sudo -u awx awx-manage dbshell -c "SELECT pg_size_pretty(pg_database_size(current_database()));"

sudo -u awx awx-manage dbshell -c "SELECT count(*) FROM pg_stat_activity;"

sudo -u awx awx-manage dbshell -c "
SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       pg_size_pretty(pg_relation_size(relid)) AS data_size,
       pg_size_pretty(pg_indexes_size(relid)) AS indexes_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
"
```

Deeper bloat / vacuum work lives in [postgres-playground](https://github.com/lennysh/postgres-playground).

---

## 4. Migrations

**When:** GUI hangs after upgrade, `tower.log` talks about migrating, or a node is out of schema sync.

```bash
sudo -u awx awx-manage showmigrations
sudo -u awx awx-manage showmigrations | grep '\[ \]'
```

`[ ]` = unapplied. Do **not** run `awx-manage migrate` unless a documented procedure or Support asked for it.

---

## 5. Redis

**When:** UI websockets dead, dispatcher stuck, `CLUSTERDOWN`, or cache looks poisoned. AAP 2.4+ often runs Redis as a cluster, not a single instance.

The unix socket talks to Redis locally and **bypasses TLS/password**. Confirm the unit/socket first — names vary (`redis`, `redis-tcp`).

```bash
systemctl list-units --type=service --all | grep -i redis
ls -l /var/run/redis/ /run/redis/ 2>/dev/null

# Read-only
redis-cli -s /var/run/redis/redis.sock ping
redis-cli -s /var/run/redis/redis.sock info replication
redis-cli -s /var/run/redis/redis.sock cluster info
redis-cli -s /var/run/redis/redis.sock cluster nodes
```

If there is no socket, `redis-cli -h 127.0.0.1 -p 6379` (and TLS/auth from `/etc/redis*` / tower settings) is the fallback.

**Mutating** — flush keys and drop cluster state. Data in Redis is cache (not the Postgres system of record) but this **will** drop live sessions, dispatcher state, and pub/sub. On a cluster, run on each node (or at least every node still in `CLUSTERDOWN`). Lab/break-glass, not a casual restart.

```bash
# as a user that can reach the socket (often root)
redis-cli -s /var/run/redis/redis.sock flushall
redis-cli -s /var/run/redis/redis.sock cluster reset hard
```

Then bounce the controller (`automation-controller-service restart`) so it re-forms the cluster membership.

---

## 6. Receptor / mesh

**When:** Execution nodes missing, jobs pending with capacity on paper, mesh TLS errors.

```bash
systemctl status receptor --no-pager
receptorctl --socket /var/run/receptor/receptor.sock status
ls -l /etc/receptor/ /var/lib/receptor/ 2>/dev/null
```

Socket path may be `/var/run/receptor/receptor.sock` or under `/var/lib/awx`. `receptorctl status --json` if you have `jq`. Cross-check capacity via the [instances API](README.md#instances-and-capacity).

---

## 7. Configuration

Read-only peeks. `settings.py` and `conf.d/*.py` contain secrets.

```bash
ls -l /etc/tower/ /etc/tower/conf.d/
# sudo -u awx less /etc/tower/settings.py

# Installer inventory (path is wherever you unpacked the bundle)
ls ansible-automation-platform-setup-*/inventory
```

---

## 8. Diagnostic bundle

From the installer directory that still has a working inventory:

```bash
./setup.sh -s
# optional: ./setup.sh -e 'target_sos_directory=/path/to/files' -s
```

Single node:

```bash
sosreport --batch
```

---

## 9. Common scenarios

### GUI not loading

```bash
sudo -u awx awx-manage showmigrations | grep '\[ \]'
automation-controller-service status
supervisorctl status
grep -i migrating /var/log/tower/tower.log | tail -20
```

### Jobs stuck in pending

Use the [instances API](README.md#instances-and-capacity), then:

```bash
receptorctl --socket /var/run/receptor/receptor.sock status
supervisorctl status
```

### Database connection issues

```bash
systemctl status postgresql --no-pager   # only if Postgres is on this host
sudo -u awx awx-manage dbshell -c "SELECT 1;"
journalctl -u postgresql -n 50 --no-pager
```
