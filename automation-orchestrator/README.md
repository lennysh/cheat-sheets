# Automation Orchestrator

Notes for [Red Hat Ansible Automation Platform automation orchestrator](https://www.redhat.com/en/technologies/management/ansible/automation-orchestrator) on OpenShift.

AO is an AAP add-on (tech preview / early access depending on build) that designs multi-mode workflows and talks to AAP as an integration. These commands are **post-install patches** on the operator-managed Deployments — typical when connecting AO to a lab/internal AAP that lives on private DNS or RFC1918.

First snippet is from internal ticket [AAP-84800](https://redhat.atlassian.net/browse/AAP-84800).

## Placeholders

| Placeholder | Replace with |
|-------------|--------------|
| `automation-orchestrator` (namespace) | Instance namespace |
| `aap.example.com` | AAP platform gateway hostname (no scheme) |
| `automation-orchestrator.example.com` | AO Route/Ingress hostname (no scheme) |

Confirm names before pasting:

```bash
oc get route,deploy -n automation-orchestrator
```

## Contents

1. [Allow OIDC to private-network IdPs / AAP](#1-allow-oidc-to-private-network-idps--aap)
2. [Allowlist AAP + AO hosts](#2-allowlist-aap--ao-hosts)
3. [Verify](#verify)
4. [Operator reconcile warning](#operator-reconcile-warning)

---

## 1. Allow OIDC to private-network IdPs / AAP

**When:** AO login or the AAP integration OAuth/OIDC handshake fails because the issuer (AAP gateway, or an IdP AAP fronts) resolves to a private, cluster, or otherwise non-public address.

**Why:** The backend fetches OIDC discovery / JWKS / token endpoints server-side. That fetch is guarded against SSRF, so hostnames that resolve to RFC1918, ULA, CGNAT, or loopback are refused unless you opt in. Lab AAP on OpenShift Routes almost always hits that guard.

```bash
oc set env deployment/automation-orchestrator-backend \
    APP_OIDC_ALLOW_PRIVATE_NETWORKS=true \
    -n automation-orchestrator

oc rollout restart deployments --selector=app.kubernetes.io/instance=automation-orchestrator
```

The restart selector rolls **every** AO Deployment in the instance (backend, worker, UI, Temporal, …). Use that after this flag so nothing is still running with the old OIDC client behavior.

Only set this in environments you trust. It widens the OIDC fetch allow-list; it is not a general “disable SSRF” switch.

---

## 2. Allowlist AAP + AO hosts

**When:** The AO UI/API rejects adding or testing an AAP integration, workers fail to call AAP, or callbacks/redirects that use AO’s own hostname are blocked.

**Why:** Outbound integration URLs are allowlisted (`APP_INTEGRATION_URL_ALLOWED_HOSTS`). Hostname only — no `https://`, no path. Backend, **worker**, and **background-worker** each have their own Deployment env. Setting the var on backend does **not** copy it to the workers; a rollout restart of the worker Deployments does not either. Patch all three in one `oc set env`.

Worker and background-worker run the same backend image and execute Temporal activities (launch/poll AAP jobs). Include AO’s own Route hostname as well as the AAP gateway.

```bash
oc set env deployment/automation-orchestrator-backend \
    deployment/automation-orchestrator-worker \
    deployment/automation-orchestrator-background-worker \
    -n automation-orchestrator \
    'APP_INTEGRATION_URL_ALLOWED_HOSTS=["aap.example.com","automation-orchestrator.example.com"]'
```

`oc set env` on a Deployment already triggers a rollout. JSON in the env value must stay a JSON **array of hostnames**; the single quotes keep the shell from eating the brackets.

Add more AAP hostnames to the JSON array if you point AO at several gateways. Keep the list tight.

---

## Verify

```bash
# Env actually landed on all three app Deployments
for d in backend worker background-worker; do
  echo "=== $d ==="
  oc set env deployment/automation-orchestrator-$d -n automation-orchestrator --list \
    | grep -E 'APP_OIDC_ALLOW_PRIVATE_NETWORKS|APP_INTEGRATION_URL_ALLOWED_HOSTS' || true
done

# Rollouts finished
oc rollout status deployment/automation-orchestrator-backend -n automation-orchestrator
oc rollout status deployment/automation-orchestrator-worker -n automation-orchestrator
oc rollout status deployment/automation-orchestrator-background-worker -n automation-orchestrator
```

Typical failure modes this set of vars addresses:

| Symptom | Likely missing var |
|---------|-------------------|
| OIDC discovery / token fetch fails; logs mention private or disallowed address | `APP_OIDC_ALLOW_PRIVATE_NETWORKS` on **backend** |
| Cannot save/test AAP integration URL in the UI | `APP_INTEGRATION_URL_ALLOWED_HOSTS` on **backend** |
| Integration saves, workflow steps that call AAP fail in a worker | Same allowlist on **worker** and **background-worker** |
| Callback / redirect to the AO Route is rejected | AO hostname missing from the allowlist |

---

## Operator reconcile warning

These Deployments are owned by the `AutomationOrchestrator` CR. The operator (or Argo CD, if the CR is GitOps-managed) can **reset env** on the next reconcile and undo `oc set env`.

If a restart or operator bump drops the vars, re-apply the snippets — or persist them on the CR if your operator version exposes extra env (not present on early CRs).

Do not commit real hostnames or tokens into this repo.
