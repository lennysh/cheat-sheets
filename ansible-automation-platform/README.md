# Ansible Automation Platform

Copy-paste diagnostics for [Red Hat Ansible Automation Platform](https://www.redhat.com/en/technologies/management/ansible).

AAP has **three install models**. The *what* (API, jobs, capacity) is the same; the *how you reach the process* is not. Pick the sheet for the model you are on, then come back here for gateway API calls that work on all three.

> Unofficial. Not Red Hat documentation. Prefer official docs for supported procedures. Commands marked **mutating** change state.

## Deployment types

| Model | What it is | Reach processes with |
|-------|------------|----------------------|
| [RPM](rpm.md) | Traditional installer on RHEL (`setup.sh`). Services via systemd / supervisord. | `systemctl`, `supervisorctl`, `automation-controller-service`, `awx-manage` as `awx` |
| [Containerized](containerized.md) | Rootless Podman on RHEL. Installer user owns the containers. | `podman`, `systemctl --user`, `journalctl CONTAINER_NAME=…` |
| [OpenShift / Operator](openshift.md) | Operator-managed pods from an `AnsibleAutomationPlatform` CR. | `oc` / `kubectl` (`get`, `logs`, `exec`, `must-gather`) |

Confirm the model before pasting. Container names, unit names, and pod names drift across 2.4 → 2.5+ (platform gateway) and across clustered Redis.

```bash
# RPM
systemctl is-active automation-controller

# Containerized (as the install user)
podman ps --format "{{.Names}}"

# OpenShift
oc get ansibleautomationplatform,pod -n aap
```

## Placeholders

| Placeholder | Replace with |
|-------------|--------------|
| `aap.example.com` | Platform gateway hostname (no scheme) |
| `$AAP_URL` | `https://aap.example.com` |
| `$AAP_TOKEN` | Personal access / OAuth token |
| `aap` (namespace) | Instance namespace (OpenShift) |
| `aap` (CR / instance name) | `AnsibleAutomationPlatform` metadata.name |
| `<JOB_ID>` / `<INSTANCE_ID>` | Numeric API ids |
| install user | User that ran the containerized installer (often `ansible` or `aap`) |

```bash
export AAP_URL=https://aap.example.com
export AAP_TOKEN='…'
```

Do not commit real hostnames, tokens, or DB passwords.

## Contents

**By install model** (same section numbers in each file):

| # | Topic | RPM | Containerized | OpenShift |
|---|-------|-----|---------------|-----------|
| 1 | Status | [rpm §1](rpm.md#1-status) | [containerized §1](containerized.md#1-status) | [openshift §1](openshift.md#1-status) |
| 2 | Logs | [rpm §2](rpm.md#2-logs) | [containerized §2](containerized.md#2-logs) | [openshift §2](openshift.md#2-logs) |
| 3 | Database | [rpm §3](rpm.md#3-database) | [containerized §3](containerized.md#3-database) | [openshift §3](openshift.md#3-database) |
| 4 | Migrations | [rpm §4](rpm.md#4-migrations) | [containerized §4](containerized.md#4-migrations) | [openshift §4](openshift.md#4-migrations) |
| 5 | Redis | [rpm §5](rpm.md#5-redis) | [containerized §5](containerized.md#5-redis) | [openshift §5](openshift.md#5-redis) |
| 6 | Receptor / mesh | [rpm §6](rpm.md#6-receptor--mesh) | [containerized §6](containerized.md#6-receptor--mesh) | [openshift §6](openshift.md#6-receptor--mesh) |
| 7 | Configuration | [rpm §7](rpm.md#7-configuration) | [containerized §7](containerized.md#7-configuration) | [openshift §7](openshift.md#7-configuration) |
| 8 | Diagnostic bundle | [rpm §8](rpm.md#8-diagnostic-bundle) | [containerized §8](containerized.md#8-diagnostic-bundle) | [openshift §8](openshift.md#8-diagnostic-bundle) |
| 9 | Common scenarios | [rpm §9](rpm.md#9-common-scenarios) | [containerized §9](containerized.md#9-common-scenarios) | [openshift §9](openshift.md#9-common-scenarios) |

**Shared (any model, via gateway):**

1. [API health](#api-health)
2. [Jobs](#jobs)
3. [Instances and capacity](#instances-and-capacity)
4. [Projects and inventories](#projects-and-inventories)
5. [Settings](#settings)
6. [Host health (VM installs)](#host-health-vm-installs)

---

## API health

**When:** UI/API is down, after upgrade, or you need a version/config snapshot.

Gateway is the front door on 2.5+. Controller still lives under `/api/controller/v2/`.

```bash
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/ping/" | jq .

curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/config/" | jq '{version, license_info, analytics_status}'
```

Username/password works (`-u "$AAP_USER:$AAP_PASS"`) but a token is less annoying to paste. `-k` is for lab TLS; drop it when the CA is trusted.

---

## Jobs

**When:** A job failed, is stuck, or you need stdout/events without the UI.

```bash
# Details + why it failed
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/jobs/<JOB_ID>/" \
  | jq '{id, name, status, job_explanation, started, finished, elapsed, execution_node, controller_node}'

# Stdout
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/jobs/<JOB_ID>/stdout/?format=txt"

# Failed events
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/jobs/<JOB_ID>/job_events/?failed=true&page_size=20" \
  | jq '.results[] | {event, task, failed, stdout}'

# Recent failures
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/jobs/?status=failed&page_size=10" \
  | jq '.results[] | {id, name, status, job_explanation, finished}'

# Job template by name
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/job_templates/?name=<TEMPLATE_NAME>" \
  | jq '.results[] | {id, name}'
```

---

## Instances and capacity

**When:** Jobs sit in pending, capacity looks wrong, or an execution node dropped off the mesh.

```bash
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/instances/" \
  | jq '.results[] | {hostname, node_type, capacity, consumed_capacity, enabled, errors}'

curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/instances/?node_type=execution" \
  | jq '.results[] | {id, hostname, capacity, consumed_capacity}'

curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/instance_groups/" \
  | jq '.results[] | {name, capacity, consumed_capacity, max_forks, max_concurrent_jobs}'
```

---

## Projects and inventories

```bash
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/projects/<PROJECT_ID>/" | jq '{id, name, scm_type, scm_url, status}'

curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/project_updates/<PROJECT_UPDATE_ID>/" \
  | jq '{id, status, failed, job_explanation}'

curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/project_updates/<PROJECT_UPDATE_ID>/stdout/?format=txt"

curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/inventories/<INVENTORY_ID>/" | jq '{id, name, hosts_with_active_failures}'
```

---

## Settings

```bash
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/settings/all/" | jq .

# Timeouts (names vary by version; jq will return null if missing)
curl -k -sS -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_URL/api/controller/v2/settings/jobs/" | jq '{DEFAULT_JOB_TIMEOUT, DEFAULT_INVENTORY_UPDATE_TIMEOUT, DEFAULT_PROJECT_UPDATE_TIMEOUT}'
```

UI-side debug flags: **Settings → Automation Execution → Troubleshooting** (tmp dir cleanup, web-request debug, receptor work retention).

---

## Host health (VM installs)

RPM and containerized share the host. OpenShift uses `oc top` / node metrics instead — see [openshift §1](openshift.md#1-status).

```bash
df -h
df -i
free -h
uptime
ss -tulpn | grep -E '443|80|5432|6379|27199'
```

Ports: `443`/`80` gateway, `5432` Postgres, `6379` Redis, `27199` receptor (mesh; confirm in `receptor.conf`).

---

## Related

- [Official AAP docs](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/)
- [Controller API](https://docs.ansible.com/automation-controller/latest/html/controllerapi/index.html)
- [postgres-playground](https://github.com/lennysh/postgres-playground) — AAP table-bloat / vacuum diagnostics
- [openshift-playground](https://github.com/lennysh/openshift-playground) — operator CR dump / restore helpers
