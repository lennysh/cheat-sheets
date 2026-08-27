# AAP — OpenShift / Operator

Operator-managed pods from an `AnsibleAutomationPlatform` CR. Everything is `oc` in the instance namespace. Pod names are `<cr-name>-<component>-…`; examples use `aap` / namespace `aap`.

Sibling sheets: [RPM](rpm.md) · [containerized](containerized.md) · [shared API](README.md)

## Placeholders (extra)

| Placeholder | Replace with |
|-------------|--------------|
| `-n aap` | Instance namespace |
| `aap` | CR `metadata.name` |
| `<task-pod>` | `…-controller-task-…` pod |
| `<redis-pod>` | `…-redis-…` pod (or `-redis-0` on a cluster) |

```bash
oc get ansibleautomationplatform,pod,route -n aap
```

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

**When:** Operator reconcile stuck, pods CrashLoop, route 503, after CSV upgrade.

```bash
oc get ansibleautomationplatform,automationcontroller,automationhub,eda -n aap
oc describe ansibleautomationplatform aap -n aap
oc get pods -n aap
oc get deploy,sts,pvc,route,svc -n aap
oc get events -n aap --sort-by='.lastTimestamp' | tail -40
```

Core CRs the operator manages: `ansibleautomationplatform` (umbrella), `automationcontroller`, `automationhub`, `eda`, plus `*backup` / `*restore`. Nested CRs can be Failed while the umbrella still looks Ready — describe both.

```bash
oc top pod -n aap
oc get csv -n aap          # operator CSV in the operator namespace may differ
oc get pods -A | grep -E 'automation-controller-operator|ansible-automation-platform'
```

---

## 2. Logs

**When:** Task/web CrashLoop, gateway 5xx, operator install role failed.

```bash
# Workload
oc logs -n aap deploy/aap-controller-web --tail=200
oc logs -n aap deploy/aap-controller-task --tail=200
oc logs -n aap deploy/aap-gateway --tail=200
oc logs -n aap -l app.kubernetes.io/name=controller-task --tail=200

# Previous crashed container
oc logs -n aap <pod> --previous --tail=200

# Operator (namespace is often ansible-automation-platform-operator or aap-operator)
oc logs -n <operator-ns> deploy/automation-controller-operator-controller-manager \
  -c automation-controller-manager -f
```

Deployment names follow `<cr-name>-controller-web` / `-controller-task` / `-gateway` / `-gateway-proxy`. If `deploy/aap-controller-task` 404s, `oc get deploy -n aap` and paste the real name.

**Mutating** — installer-role verbosity (can leak secrets; lab only):

```yaml
# on the AnsibleAutomationPlatform CR
spec:
  controller:
    no_log: false
metadata:
  annotations:
    ansible.sdk.operatorframework.io/verbosity: "4"
```

---

## 3. Database

**When:** Web/task pods crashloop on migrate or `could not connect to server`.

Managed Postgres is a pod/statefulset in the same namespace. External DB: `psql` from a jump host, not `oc exec`.

```bash
oc get pods,sts,pvc -n aap | grep -iE 'postgres|postgresql'
oc logs -n aap <postgres-pod> --tail=50

# Via controller (uses the app’s DB settings)
oc exec -n aap <task-pod> -- awx-manage dbshell -c "SELECT version();"
oc exec -n aap <task-pod> -- awx-manage dbshell -c "SELECT pg_size_pretty(pg_database_size(current_database()));"
oc exec -n aap <task-pod> -- awx-manage dbshell -c "SELECT count(*) FROM pg_stat_activity;"
```

Table sizes (same SQL as the other sheets):

```bash
oc exec -n aap <task-pod> -- awx-manage dbshell -c "
SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       pg_size_pretty(pg_relation_size(relid)) AS data_size,
       pg_size_pretty(pg_indexes_size(relid)) AS indexes_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
"
```

Pick the task pod:

```bash
oc get pod -n aap -o name | grep controller-task
```

---

## 4. Migrations

**When:** UI hangs after operator upgrade, task pod logs “migrating”, web CrashLoop on schema.

```bash
oc exec -n aap <task-pod> -- awx-manage showmigrations
oc exec -n aap <task-pod> -- awx-manage showmigrations | grep '\[ \]'
oc logs -n aap <task-pod> | grep -i migrating | tail -20
```

`[ ]` = unapplied. Do **not** run `awx-manage migrate` in the pod unless a documented procedure or Support asked for it — the operator owns schema during reconcile.

---

## 5. Redis

**When:** UI websockets dead, dispatcher stuck, `CLUSTERDOWN`, or cache looks poisoned.

Standalone Redis is the operator default; clustered Redis is 6 replicas. Switching mode on an existing instance is **not supported**. Names: `<cr-name>-redis` or `<cr-name>-redis-0` … `-redis-5`.

The unix socket inside the Redis container **bypasses TLS/password** (same trick as containerized).

```bash
oc get pods,sts,deploy -n aap | grep -i redis

oc exec -n aap <redis-pod> -- redis-cli -s /run/redis/redis.sock ping
oc exec -n aap <redis-pod> -- redis-cli -s /run/redis/redis.sock info replication
oc exec -n aap <redis-pod> -- redis-cli -s /run/redis/redis.sock cluster info
oc exec -n aap <redis-pod> -- redis-cli -s /run/redis/redis.sock cluster nodes
```

If the socket path 404s, `oc exec -n aap <redis-pod> -- ls -l /run/redis /var/run/redis`.

**Mutating** — flush keys and drop cluster state. Redis is cache, not Postgres, but this **drops** sessions, dispatcher state, and pub/sub. On clustered Redis, run on each replica. Lab/break-glass. The operator may recreate pods; re-check `cluster info` after.

```bash
oc exec -n aap <redis-pod> -- redis-cli -s /run/redis/redis.sock flushall
oc exec -n aap <redis-pod> -- redis-cli -s /run/redis/redis.sock cluster reset hard
```

Then roll the consumers:

```bash
oc rollout restart deploy -n aap -l app.kubernetes.io/component=controller
oc rollout status deploy/aap-controller-web -n aap
oc rollout status deploy/aap-controller-task -n aap
```

---

## 6. Receptor / mesh

**When:** Execution nodes missing, jobs pending, hop nodes unhealthy.

On OpenShift, receptor runs **in the controller pods** (not a separate `receptor` container like containerized). Mesh to remote execution nodes is still receptor; hop/execution VMs are the RPM/containerized receptor commands on *those* hosts.

```bash
oc exec -n aap <task-pod> -- receptorctl status
oc logs -n aap <task-pod> | grep -i receptor | tail -50
```

Cross-check [instances API](README.md#instances-and-capacity). Remote execution node: SSH there and use the [RPM](rpm.md#6-receptor--mesh) or [containerized](containerized.md#6-receptor--mesh) receptor section.

---

## 7. Configuration

Desired state is the CR, not a file on disk. Secrets are K8s Secrets (do not `oc get secret -o yaml` into git).

```bash
oc get ansibleautomationplatform aap -n aap -o yaml
oc get automationcontroller -n aap -o yaml
oc describe route -n aap
oc get secret -n aap
```

Extra settings belong on the CR (`extra_settings` / component spec), not `oc set env` on the Deployment — the operator will revert naked env patches on reconcile.

---

## 8. Diagnostic bundle

`must-gather` image tag follows the AAP version (`-25`, `-26`, `-27`, …).

```bash
# cluster-wide
oc adm must-gather \
  --image=registry.redhat.io/ansible-automation-platform-26/aap-must-gather-rhel9 \
  --dest-dir ./must-gather

# one namespace
oc adm must-gather \
  --image=registry.redhat.io/ansible-automation-platform-26/aap-must-gather-rhel9 \
  --dest-dir ./must-gather \
  -- /usr/bin/ns-gather aap

tar cvaf must-gather.tar.gz must-gather.local.*/
```

AAP 2.5 used `…-platform-25/aap-must-gather-rhel8` in older docs; 2.6+ is `rhel9`. Confirm the image for your CSV.

---

## 9. Common scenarios

### GUI not loading / route 503

```bash
oc get route,pod -n aap
oc describe ansibleautomationplatform aap -n aap | sed -n '/Conditions/,/Events/p'
oc logs -n aap deploy/aap-gateway-proxy --tail=80
oc exec -n aap <task-pod> -- awx-manage showmigrations | grep '\[ \]'
```

### Jobs stuck in pending

[Instances API](README.md#instances-and-capacity), then receptor on the task pod (§6) and `oc get pods -n aap` for job pods if “Release Receptor Work” is off in Troubleshooting settings.

### Database connection issues

```bash
oc get pods -n aap | grep -i postgres
oc logs -n aap <postgres-pod> --tail=50
oc exec -n aap <task-pod> -- awx-manage dbshell -c "SELECT 1;"
```

### Operator keeps resetting a patch

You edited a Deployment and it snapped back. Put the change on the `AnsibleAutomationPlatform` CR (or stop fighting the operator). Same warning as Automation Orchestrator env vars.
