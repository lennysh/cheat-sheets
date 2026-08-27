# AAP — Containerized

Rootless Podman on RHEL. Log in as the **install user** (the account that ran the installer — often `ansible` or `aap`). Commands below assume that user; `sudo` is usually the wrong namespace.

Sibling sheets: [RPM](rpm.md) · [OpenShift](openshift.md) · [shared API](README.md)

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

**When:** A container exited, after reboot (user lingering / linger), or installer rerun.

```bash
podman ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
podman ps --all --format "{{.Names}}"

systemctl --user status automation-controller-web automation-controller-task \
  automation-gateway automation-gateway-proxy postgresql redis receptor --no-pager
```

Each component is a user systemd unit that `podman start`s the container. `systemctl --user restart <name>` is the supported bounce; `podman restart` fights systemd.

| Group | Typical container name |
|-------|------------------------|
| Controller | `automation-controller-web`, `automation-controller-task`, `automation-controller-rsyslog` |
| Gateway | `automation-gateway`, `automation-gateway-proxy` |
| Hub | `automation-hub-api`, `automation-hub-content`, `automation-hub-web`, `automation-hub-worker-<n>` |
| EDA | `automation-eda-api`, `automation-eda-web`, `automation-eda-daphne`, `automation-eda-scheduler`, `automation-eda-worker-<n>`, `automation-eda-activation-worker-<n>` |
| Data plane | `postgresql`, `receptor`, `redis-<suffix>` (often `redis-tcp`) |

```bash
podman stats --no-stream
podman inspect <container_name> | jq '.[0] | {State: .State.Status, Health: .State.Health, Mounts: [.Mounts[].Source]}'
```

---

## 2. Logs

**When:** Need a container’s stdout or a time window. Containerized AAP logs through journald **and** `podman logs`.

```bash
journalctl CONTAINER_NAME=<container_name> --no-pager
journalctl CONTAINER_NAME=automation-controller-task -n 200 --no-pager

podman logs --tail 200 <container_name>
podman logs -f <container_name>
podman logs -t --since="2026-08-01T00:00:00" --until="2026-08-01T23:59:59" automation-controller-task
```

Useful names: `automation-controller-task`, `automation-controller-web`, `automation-gateway`, `automation-gateway-proxy`, `postgresql`, `redis-tcp` (or `redis-<suffix>`), `receptor`.

---

## 3. Database

**When:** Controller containers restart-loop, jobs fail with DB errors, or you need size / connections.

```bash
podman ps | grep postgres
podman logs --tail 50 postgresql

podman exec postgresql psql -U awx -d awx -c "SELECT version();"
podman exec postgresql psql -U awx -d awx -c "SELECT pg_size_pretty(pg_database_size('awx'));"
podman exec postgresql psql -U awx -d awx -c "SELECT count(*) FROM pg_stat_activity WHERE datname = 'awx';"

podman exec postgresql psql -U awx -d awx -c "
SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       pg_size_pretty(pg_relation_size(relid)) AS data_size,
       pg_size_pretty(pg_indexes_size(relid)) AS indexes_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
"
```

Replace `postgresql` / `awx` if your inventory used different names. External DB: `psql` from the host with those settings, not `podman exec`.

Same queries via the controller (picks up container settings.py):

```bash
podman exec automation-controller-task awx-manage dbshell -c "SELECT 1;"
```

---

## 4. Migrations

**When:** UI hangs after upgrade, task container logs “migrating”.

```bash
podman exec automation-controller-task awx-manage showmigrations
podman exec automation-controller-task awx-manage showmigrations | grep '\[ \]'
```

`[ ]` = unapplied. Do **not** run `awx-manage migrate` unless a documented procedure or Support asked for it.

---

## 5. Redis

**When:** UI websockets dead, dispatcher stuck, `CLUSTERDOWN`, or cache looks poisoned.

The unix socket inside the Redis container **bypasses TLS/password**. Names are `redis-tcp` or `redis-<suffix>` — list first.

```bash
podman ps --format "{{.Names}}" | grep redis
systemctl --user status redis --no-pager

podman exec redis-tcp redis-cli -s /run/redis/redis.sock ping
podman exec redis-tcp redis-cli -s /run/redis/redis.sock info replication
podman exec redis-tcp redis-cli -s /run/redis/redis.sock cluster info
podman exec redis-tcp redis-cli -s /run/redis/redis.sock cluster nodes
```

**Mutating** — flush keys and drop cluster state. Redis is cache, not Postgres, but this **drops** sessions, dispatcher state, and pub/sub. On a multi-node Redis cluster, run on each Redis container. Lab/break-glass.

```bash
podman exec redis-tcp redis-cli -s /run/redis/redis.sock flushall
podman exec redis-tcp redis-cli -s /run/redis/redis.sock cluster reset hard
```

Then bounce the consumers so they rejoin:

```bash
systemctl --user restart automation-controller-web automation-controller-task automation-gateway
```

---

## 6. Receptor / mesh

**When:** Execution nodes missing, jobs pending with capacity on paper.

```bash
podman logs --tail 100 receptor
systemctl --user status receptor --no-pager

# host-side socket (install-user home)
receptorctl --socket "$HOME/aap/receptor/run/receptor.sock" status

# inside the container — confirm the mount first
podman inspect receptor | jq -r '.[].Mounts[] | "\(.Source) -> \(.Destination)"'
podman exec receptor receptorctl --socket /run/receptor/receptor.sock status
```

If the in-container path 404s, use the `Destination` from inspect. Cross-check [instances API](README.md#instances-and-capacity).

---

## 7. Configuration

Layout under the install user’s home (`$HOME/aap/`):

| Path | What |
|------|------|
| `$HOME/aap/controller/etc/` | `settings.py`, TLS, conf.d |
| `$HOME/aap/controller/data/` | jobs / projects / rsyslog |
| `$HOME/aap/receptor/etc/receptor.conf` | mesh |
| `$HOME/aap/tls/` | installer-generated certs |
| `$HOME/.config/systemd/user/` | per-container units |
| `$HOME/.local/share/containers/storage/volumes/` | Podman volumes |

```bash
ls "$HOME/aap/controller/etc/" "$HOME/aap/receptor/etc/"
podman inspect automation-controller-task | jq '.[0].Config.Env'
podman volume ls
```

Those files contain secrets. Do not copy them into this repo.

---

## 8. Diagnostic bundle

From the containerized installer directory (inventory still valid):

```bash
ansible-playbook -i inventory ansible.containerized_installer.log_gathering

# extra sos plugins as root, as the install user name
sudo sos report -e aap_containerized -k aap_containerized.username=<install_user>
```

---

## 9. Common scenarios

### GUI not loading

```bash
podman ps -a | grep -E 'controller|gateway'
podman exec automation-controller-task awx-manage showmigrations | grep '\[ \]'
podman logs --tail 80 automation-controller-task | grep -i migrating
podman logs --tail 80 automation-gateway-proxy
```

### Jobs stuck in pending

[Instances API](README.md#instances-and-capacity), then:

```bash
podman logs --tail 80 receptor
podman logs --tail 80 automation-controller-task
```

### Database connection issues

```bash
podman ps -a | grep postgres
podman logs --tail 50 postgresql
podman exec automation-controller-task awx-manage dbshell -c "SELECT 1;"
```

### User lingering / containers dead after logout

Rootless Podman dies if the install user has no session. `loginctl enable-linger <install_user>` is the usual fix (mutating; needs root).
