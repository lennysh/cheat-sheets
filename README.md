# Cheat sheets

Personal copy-paste notes — commands I actually run, with just enough context to remember *when* and *why*.

Companion to the playground repos (labs and demos), not a replacement for them.

> Unofficial. Not Red Hat documentation. Prefer official docs for supported procedures.

## Sheets

| Topic | Path | What's in it |
|-------|------|----------------|
| Automation Orchestrator | [automation-orchestrator/](automation-orchestrator/) | AAP integration env vars: private-network OIDC, URL allowlists, rollouts |
| Ansible | [ansible/](ansible/) | Playbook Jinja2 filters, tests, and task/play keywords |
| AAP | [ansible-automation-platform/](ansible-automation-platform/) | Diagnostics by install model (RPM, containerized, OpenShift) plus shared gateway API |
| OpenShift | — | *planned* |

## Layout

One folder per product or topic. The folder `README.md` is the cheat sheet (or an index if that topic grows more than one file).

```text
cheat-sheets/
├── README.md                      # this index
├── ansible/
│   └── README.md                  # filters / tests / keywords
├── ansible-automation-platform/
│   ├── README.md                  # index + shared API
│   ├── rpm.md
│   ├── containerized.md
│   └── openshift.md
└── automation-orchestrator/
    └── README.md
```

Keep hostnames and namespaces as placeholders (`aap.example.com`, `-n automation-orchestrator`). Swap in lab values when you paste.

## Adding a sheet

1. Create `topic-name/README.md` (kebab-case folder, matching the other playgrounds).
2. Add a row to the **Sheets** table above.
3. If the topic needs more than one page, keep the folder README as a short index and add sibling `.md` files.

## Related repos

- [ansible-playground](https://github.com/lennysh/ansible-playground) — playbooks and AAP Config-as-Code
- [argocd-playground](https://github.com/lennysh/argocd-playground) — Argo CD GitOps for AAP, Automation Orchestrator, and related apps on OpenShift
- [eda-playground](https://github.com/lennysh/eda-playground) — Event-Driven Ansible
- [openshift-playground](https://github.com/lennysh/openshift-playground) — OpenShift / AAP operator utilities
- [postgres-playground](https://github.com/lennysh/postgres-playground) — PostgreSQL diagnostics
- [opa-playground](https://github.com/lennysh/opa-playground) — AAP OPA policies
