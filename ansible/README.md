# Ansible playbook filters

Reference for **Jinja2 filters**, **tests**, and **playbook/task keywords** beyond basic modules and roles. Assumes you already know how to run playbooks, use modules, and structure roles.

> Unofficial. Not Red Hat documentation. Prefer official docs for supported procedures.

> **Tip:** Use `ansible.builtin.debug` with `var:` to inspect filter output on real data. Pipe expressions in `{{ }}` or assign with `set_fact`.

### How to read examples

| Column | Meaning |
|--------|---------|
| **Before** | Input variable value (YAML/JSON) |
| **Expression** | Jinja2 filter chain in a playbook |
| **After** | Result Ansible evaluates to |

Filter names use Ansible’s canonical spelling: `selectattr` / `rejectattr` (no underscore). Aliases like `eq` for `equalto` are noted where applicable.

---

## Table of Contents

1. [Quick reference: tests for select/reject/selectattr/rejectattr](#quick-reference-tests-for-selectrejectselectattrrejectattr)
2. [List & sequence filters](#list--sequence-filters)
3. [Dictionary filters](#dictionary-filters)
4. [Set & list math filters](#set--list-math-filters)
5. [String filters](#string-filters)
6. [Type, default, and safety filters](#type-default-and-safety-filters)
7. [Serialization filters (JSON/YAML/INI)](#serialization-filters-jsonyamlini)
8. [Path & file filters](#path--file-filters)
9. [Math & numeric filters](#math--numeric-filters)
10. [Encoding, hashing, and secrets](#encoding-hashing-and-secrets)
11. [Grouping, batching, and advanced list ops](#grouping-batching-and-advanced-list-ops)
12. [Network filters (ansible.utils)](#network-filters-ansibleutils)
13. [Debugging & introspection filters](#debugging--introspection-filters)
14. [Jinja2 tests (standalone)](#jinja2-tests-standalone)
15. [Task & play keywords (async, delegation, control flow)](#task--play-keywords-async-delegation-control-flow)
16. [Conditionals: failed_when, changed_when, until](#conditionals-failed_when-changed_when-until)
17. [Loops, registers, and facts](#loops-registers-and-facts)
18. [Execution strategy, serial, and throttle](#execution-strategy-serial-and-throttle)
19. [Sources](#sources)

---

## Quick reference: tests for select/reject/selectattr/rejectattr

These four filters share the same **test** vocabulary. Use a test name as the second argument; omit the third argument for truthiness checks.

| Test | Aliases | Meaning |
|------|---------|---------|
| `equalto` | `eq`, `==` | Equal to value |
| `ne` | `!=` | Not equal |
| `gt`, `ge`, `lt`, `le` | `>`, `>=`, `<`, `<=` | Numeric/string compare |
| `in` | — | Value in sequence |
| `match` | — | Regex matches **start** of string |
| `search` | — | Regex found **anywhere** |
| `regex` | — | Alias for `match` in some contexts |
| `defined` | — | Attribute/key exists |
| `undefined` | — | Attribute/key missing |
| `none` | — | Value is `None` / null |
| *(omitted)* | — | Truthy attribute/value |

---

## List & sequence filters

### `selectattr` / `rejectattr`

Filter a **list of dicts** by an attribute. `rejectattr` keeps items where the test **fails**.

#### `selectattr` — equalto

**Before:**
```yaml
servers:
  - { name: web1, env: prod, port: 443 }
  - { name: web2, env: dev,  port: 8080 }
  - { name: db1,  env: prod, port: 5432 }
```

**Expression:**
```yaml
{{ servers | selectattr('env', 'equalto', 'prod') | list }}
```

**After:**
```yaml
- { name: web1, env: prod, port: 443 }
- { name: db1,  env: prod, port: 5432 }
```

#### `rejectattr` — equalto

**Expression:**
```yaml
{{ servers | rejectattr('env', 'equalto', 'prod') | list }}
```

**After:**
```yaml
- { name: web2, env: dev, port: 8080 }
```

#### `selectattr` — truthiness (no test name)

**Before:**
```yaml
hosts:
  - { name: a, enabled: true }
  - { name: b, enabled: false }
  - { name: c, enabled: true }
```

**Expression:**
```yaml
{{ hosts | selectattr('enabled') | map(attribute='name') | list }}
```

**After:**
```yaml
['a', 'c']
```

#### `selectattr` — `search` (substring / regex)

**Before:**
```yaml
users:
  - { login: admin@corp.com, role: admin }
  - { login: bob@corp.com,   role: user }
  - { login: svc-bot,        role: service }
```

**Expression:**
```yaml
{{ users | selectattr('login', 'search', '@corp.com') | list }}
```

**After:**
```yaml
- { login: admin@corp.com, role: admin }
- { login: bob@corp.com,   role: user }
```

#### `selectattr` — nested attribute (`attr`)

**Before:**
```yaml
vms:
  - { name: vm1, meta: { zone: us-east-1 } }
  - { name: vm2, meta: { zone: eu-west-1 } }
```

**Expression:**
```yaml
{{ vms | selectattr('meta.zone', 'equalto', 'us-east-1') | list }}
```

**After:**
```yaml
- { name: vm1, meta: { zone: us-east-1 } }
```

#### `selectattr` — `defined` / `undefined`

**Before:**
```yaml
items:
  - { id: 1, comment: "ok" }
  - { id: 2 }
  - { id: 3, comment: "" }
```

**Expression:**
```yaml
{{ items | selectattr('comment', 'defined') | list }}
```

**After:**
```yaml
- { id: 1, comment: "ok" }
- { id: 3, comment: "" }
```

```yaml
{{ items | rejectattr('comment', 'defined') | list }}
```

**After:**
```yaml
- { id: 2 }
```

#### `selectattr` — numeric `gt`

**Before:**
```yaml
disks:
  - { mount: /,  size_gb: 50 }
  - { mount: /var, size_gb: 200 }
```

**Expression:**
```yaml
{{ disks | selectattr('size_gb', 'gt', 100) | list }}
```

**After:**
```yaml
- { mount: /var, size_gb: 200 }
```

---

### `select` / `reject`

Same test vocabulary, but operate on **plain list values** (not dict attributes).

**Before:**
```yaml
tags: [prod, dev, prod, staging, prod]
```

**Expression:**
```yaml
{{ tags | select('equalto', 'prod') | list }}
```

**After:**
```yaml
[prod, prod, prod]
```

```yaml
{{ tags | reject('equalto', 'prod') | unique | list }}
```

**After:**
```yaml
[dev, staging]
```

---

### `map`

Extract or transform each element.

#### `map(attribute='...')`

**Before:**
```yaml
vms:
  - { name: vm1, ip: 10.0.0.1 }
  - { name: vm2, ip: 10.0.0.2 }
```

**Expression:**
```yaml
{{ vms | map(attribute='ip') | list }}
```

**After:**
```yaml
['10.0.0.1', '10.0.0.2']
```

#### `map('combine', {...})`

**Before:**
```yaml
users:
  - { name: alice, active: true }
  - { name: bob,   active: false }
```

**Expression:**
```yaml
{{ users | map('combine', {'notified': false}) | list }}
```

**After:**
```yaml
- { name: alice, active: true,  notified: false }
- { name: bob,   active: false, notified: false }
```

#### `map('extract', groups, 'group_name')`

**Before:**
```yaml
group_names: [web, db]
groups:
  web: [host1, host2]
  db:  [host3]
```

**Expression:**
```yaml
{{ group_names | map('extract', groups) | flatten | list }}
```

**After:**
```yaml
[host1, host2, host3]
```

---

### `first` / `last`

**Before:**
```yaml
queue: [alpha, beta, gamma]
```

| Expression | After |
|------------|-------|
| `{{ queue \| first }}` | `alpha` |
| `{{ queue \| last }}` | `gamma` |

---

### `min` / `max` / `sort`

**Before:**
```yaml
scores: [42, 7, 99, 15]
```

| Expression | After |
|------------|-------|
| `{{ scores \| min }}` | `7` |
| `{{ scores \| max }}` | `99` |
| `{{ scores \| sort }}` | `[7, 15, 42, 99]` |
| `{{ scores \| sort(reverse=true) }}` | `[99, 42, 15, 7]` |

**Before (dicts):**
```yaml
pods:
  - { name: a, cpu: 2 }
  - { name: b, cpu: 0.5 }
```

**Expression:**
```yaml
{{ pods | sort(attribute='cpu', reverse=true) | first }}
```

**After:**
```yaml
{ name: a, cpu: 2 }
```

---

### `unique`

**Before:**
```yaml
ids: [1, 2, 2, 3, 1, 3]
```

**Expression:**
```yaml
{{ ids | unique | list }}
```

**After:**
```yaml
[1, 2, 3]
```

---

### `flatten`

**Before:**
```yaml
nested: [[a, b], [c], [], [d, e]]
```

**Expression:**
```yaml
{{ nested | flatten | list }}
```

**After:**
```yaml
[a, b, c, d, e]
```

**With levels:**
```yaml
deep: [1, [2, [3, 4]]]
# {{ deep | flatten(levels=1) }}  →  [1, 2, [3, 4]]
```

---

### `zip` / `zip_longest`

**Before:**
```yaml
names: [web, db]
ports: [80, 5432]
```

**Expression:**
```yaml
{{ names | zip(ports) | list }}
```

**After:**
```yaml
- [web, 80]
- [db, 5432]
```

**Expression (unequal lengths, fill):**
```yaml
{{ names | zip_longest(ports, fillvalue=0) | list }}
```

---

### `slice`

Chunk a list into sublists of size N.

**Before:**
```yaml
hosts: [h1, h2, h3, h4, h5]
```

**Expression:**
```yaml
{{ hosts | slice(2) | list }}
```

**After:**
```yaml
- [h1, h2]
- [h3, h4]
- [h5]
```

---

### `reverse` / `shuffle` / `random`

**Before:**
```yaml
order: [1, 2, 3]
```

| Expression | After (example) |
|------------|-----------------|
| `{{ order \| reverse \| list }}` | `[3, 2, 1]` |
| `{{ order \| shuffle \| list }}` | non-deterministic order |
| `{{ order \| random }}` | one arbitrary element |

---

### `union` (list merge, unique)

**Before:**
```yaml
a: [1, 2, 3]
b: [3, 4, 5]
```

**Expression:**
```yaml
{{ a | union(b) | list }}
```

**After:**
```yaml
[1, 2, 3, 4, 5]
```

---

## Dictionary filters

### `combine`

Merge dicts; later keys win. Use `recursive=true` for nested merge.

**Before:**
```yaml
defaults:
  app:
    port: 8080
    ssl: false
override:
  app:
    ssl: true
```

**Expression:**
```yaml
{{ defaults | combine(override) }}
```

**After:**
```yaml
app:
  port: 8080
  ssl: true
```

**Expression (recursive):**
```yaml
{{ defaults | combine(override, recursive=true) }}
```

**After:** same as above when only `ssl` is overridden under `app`.

**List-right syntax:**
```yaml
{{ defaults | combine(override, list_merge='append') }}
```

---

### `dict2items` / `items2dict`

**Before:**
```yaml
mounts:
  /var: { size: 50G }
  /tmp: { size: 10G }
```

**Expression:**
```yaml
{{ mounts | dict2items }}
```

**After:**
```yaml
- key: /var
  value: { size: 50G }
- key: /tmp
  value: { size: 10G }
```

**Round-trip with key rename:**
```yaml
{{ mounts | dict2items(key_name='mount', value_name='opts') }}
```

**Expression (`items2dict`):**
```yaml
{{ mounts | dict2items | selectattr('key', 'search', '^/var') | items2dict }}
```

**After:**
```yaml
/var: { size: 50G }
```

---

### `dictsort`

**Before:**
```yaml
counts: { zebra: 1, alpha: 3, beta: 2 }
```

**Expression:**
```yaml
{{ counts | dictsort }}
```

**After:**
```yaml
- [alpha, 3]
- [beta, 2]
- [zebra, 1]
```

By value: `{{ counts | dictsort(true) }}` sorts by value descending.

---

## Set & list math filters

| Filter | Description |
|--------|-------------|
| `intersect` | Elements in both lists |
| `difference` | In first, not in second |
| `symmetric_difference` | In either, not both |

**Before:**
```yaml
installed: [httpd, mariadb, php]
required:  [httpd, php, redis]
```

| Expression | After |
|------------|-------|
| `{{ installed \| intersect(required) \| list }}` | `[httpd, php]` |
| `{{ installed \| difference(required) \| list }}` | `[mariadb]` |
| `{{ installed \| symmetric_difference(required) \| list }}` | `[mariadb, redis]` |

---

## String filters

### `split` / `join`

**Before:**
```yaml
csv: "10.1.1.1,10.1.1.2,10.1.1.3"
```

**Expression:**
```yaml
{{ csv.split(',') }}
# or
{{ csv | split(',') }}
```

**After:**
```yaml
['10.1.1.1', '10.1.1.2', '10.1.1.3']
```

**Expression:**
```yaml
{{ ['a', 'b', 'c'] | join(':') }}
```

**After:**
```yaml
"a:b:c"
```

---

### `regex_replace` / `regex_search` / `regex_findall`

**Before:**
```yaml
line: "server_name=web01.example.com;"
```

| Expression | After |
|------------|-------|
| `{{ line \| regex_replace('^server_name=', '') }}` | `web01.example.com;` |
| `{{ line \| regex_search('([a-z0-9-]+)\\.example') }}` | match group or `None` |
| `{{ 'a1 b22 c333' \| regex_findall('\\d+') }}` | `['1', '22', '333']` |

---

### `replace` / `trim` / `lower` / `upper` / `capitalize`

**Before:**
```yaml
msg: "  Hello WORLD  "
```

| Expression | After |
|------------|-------|
| `{{ msg \| trim }}` | `Hello WORLD` |
| `{{ msg \| trim \| lower }}` | `hello world` |
| `{{ 'foo-bar' \| replace('-', '_') }}` | `foo_bar` |

---

### `quote` / `to_uuid`

**Before:**
```yaml
user: O'Brien
```

**Expression:**
```yaml
{{ user | quote }}
```

**After:**
```yaml
"O'Brien"
```

#### `to_uuid` — namespaced UUIDv5 (deterministic)

Generates the same UUID every time for the same input string and namespace. Useful for stable IDs in templates (e.g. VM names, hostnames) without storing a mapping.

**Before:**
```yaml
hostname: web01.example.com
```

**Expression (default Ansible namespace `361E6D51-FAEC-444A-9079-341386DA8E2E`):**
```yaml
{{ hostname | to_uuid }}
```

**After:**
```yaml
4495a025-92f7-5f8a-914f-86de0f3ec120
```

**Expression (custom namespace):**
```yaml
{{ hostname | to_uuid(namespace='11111111-2222-3333-4444-555555555555') }}
```

**After:**
```yaml
9cc5a212-0655-5171-8245-28609dd2a12b
```

> Same `hostname` + same `namespace` always yields the same UUID. Changing either produces a different UUID.

---

### `hash` / `checksum`

**Before:**
```yaml
payload: "configure me"
```

| Expression | After (examples) |
|------------|------------------|
| `{{ payload \| hash('sha256') }}` | hex digest string |
| `{{ payload \| checksum }}` | sha1 checksum (legacy default) |

---

## Type, default, and safety filters

### `default` / `d`

**Before:** variable `optional_port` is **undefined**

**Expression:**
```yaml
{{ optional_port | default(8080) }}
```

**After:**
```yaml
8080
```

**Truthy default only when undefined/false:**
```yaml
{{ flag | default(true, true) }}   # second arg: boolean default
```

---

### `mandatory`

**Before:** `api_key` undefined

**Expression:**
```yaml
{{ api_key | mandatory }}
```

**After:** task/play **fails** with clear error if undefined.

---

### `bool` / `int` / `float` / `string`

**Before:**
```yaml
raw: "yes"
```

| Expression | After |
|------------|-------|
| `{{ raw \| bool }}` | `true` |
| `{{ '42' \| int }}` | `42` |
| `{{ 3.14 \| string }}` | `"3.14"` |

`bool` treats `yes`, `on`, `1`, `true` (case-insensitive) as true.

---

### `ternary`

**Before:**
```yaml
dry_run: true
```

**Expression:**
```yaml
{{ dry_run | ternary('--check', '') }}
```

**After:**
```yaml
"--check"
```

---

### `ternary` with `is defined`

```yaml
{{ my_var is defined | ternary(my_var, 'fallback') }}
```

---

## Serialization filters (JSON/YAML/INI)

### `to_json` / `from_json`

**Before:**
```yaml
data: { hosts: [a, b], ssl: true }
```

**Expression:**
```yaml
{{ data | to_json }}
```

**After:**
```yaml
'{"hosts": ["a", "b"], "ssl": true}'
```

**Round-trip:**
```yaml
{{ '{"x": 1}' | from_json }}
# → { x: 1 }
```

---

### `to_yaml` / `to_nice_yaml` / `from_yaml`

**Before:**
```yaml
config: { listen: 443, backends: [10.0.0.1, 10.0.0.2] }
```

**Expression:**
```yaml
{{ config | to_nice_yaml(indent=2) }}
```

**After:**
```yaml
"listen: 443\nbackends:\n  - 10.0.0.1\n  - 10.0.0.2\n"
```

---

### `to_ini`

**Before:**
```yaml
section:
  database:
    host: db1
    port: 5432
```

**Expression:**
```yaml
{{ section | to_ini }}
```

**After:**
```ini
[database]
host = db1
port = 5432
```

---

## Path & file filters

| Filter | Before | Expression | After |
|--------|--------|------------|-------|
| `basename` | `/etc/nginx/nginx.conf` | `{{ path \| basename }}` | `nginx.conf` |
| `dirname` | `/etc/nginx/nginx.conf` | `{{ path \| dirname }}` | `/etc/nginx` |
| `expanduser` | `~/bin` | `{{ path \| expanduser }}` | `/home/user/bin` |
| `realpath` | `../etc` (must exist on controller) | `{{ path \| realpath }}` | resolved absolute path |
| `relpath` | abs paths | `{{ f \| relpath(base) }}` | relative path |

### `fileglob`

**Expression (on controller):**
```yaml
{{ '/etc/ansible/*.cfg' | fileglob }}
```

**After:** list of matching paths.

---

## Math & numeric filters

**Before:**
```yaml
n: -3.7
```

| Expression | After |
|------------|-------|
| `{{ n \| abs }}` | `3.7` |
| `{{ n \| round(0, 'ceil') \| int }}` | `-3` |
| `{{ 10 \| log(2) }}` | ~`3.32` |
| `{{ 2 \| pow(8) }}` | `256` |
| `{{ 5 \| root(2) }}` | `~2.236` |

### `human_readable` / `human_to_bytes`

| Expression | After |
|------------|-------|
| `{{ 1048576 \| human_readable }}` | `1.0 MB` |
| `{{ '1.5 GiB' \| human_to_bytes }}` | byte integer |

---

## Encoding, hashing, and secrets

| Filter | Example | Notes |
|--------|---------|-------|
| `b64encode` | `{{ 'secret' \| b64encode }}` | bytes/string → base64 |
| `b64decode` | `{{ encoded \| b64decode }}` | reverse |
| `password_hash` | `{{ 'plain' \| password_hash('sha512') }}` | for `/etc/shadow` style hashes |

**Before:**
```yaml
token: "api-key-123"
```

**Expression:**
```yaml
{{ token | b64encode }}
```

**After:**
```yaml
YXBpLWtleS0xMjM=
```

---

## Grouping, batching, and advanced list ops

### `groupby`

**Before:**
```yaml
servers:
  - { name: s1, env: prod }
  - { name: s2, env: dev }
  - { name: s3, env: prod }
```

**Expression:**
```yaml
{{ servers | groupby('env') | list }}
```

**After:**
```yaml
- [prod, [{ name: s1, env: prod }, { name: s3, env: prod }]]
- [dev,  [{ name: s2, env: dev }]]
```

---

### `batch`

**Before:**
```yaml
items: [1, 2, 3, 4, 5]
```

**Expression:**
```yaml
{{ items | batch(2, fillvalue=0) | list }}
```

**After:**
```yaml
- [1, 2]
- [3, 4]
- [5, 0]
```

---

### `product` / `permutations` / `combinations`

**Before:**
```yaml
colors: [red, blue]
sizes:  [S, M]
```

**Expression:**
```yaml
{{ colors | product(sizes) | list }}
```

**After:**
```yaml
- [red, S]
- [red, M]
- [blue, S]
- [blue, M]
```

```yaml
{{ [1, 2, 3] | combinations(2) | list }}
# → [[1,2], [1,3], [2,3]]
```

---

## Network filters (ansible.utils)

> Requires `ansible.utils` collection. Install: `ansible-galaxy collection install ansible.utils`

### `ipaddr` / `ipv4` / `ipv6`

**Before:**
```yaml
addrs: ['192.168.1.10/24', '2001:db8::1/64', 'not-an-ip', '10.0.0.1']
```

**Expression:**
```yaml
{{ addrs | ansible.utils.ipaddr('private') | list }}
```

**After:**
```yaml
['192.168.1.10/24', '10.0.0.1']
```

**Expression:**
```yaml
{{ '192.168.1.10/24' | ansible.utils.ipaddr('address') }}
```

**After:**
```yaml
'192.168.1.10'
```

**Expression:**
```yaml
{{ '192.168.1.10/24' | ansible.utils.ipaddr('net') }}
```

**After:**
```yaml
'192.168.1.0/24'
```

### `ipsubnet` / `nthhost`

```yaml
{{ '192.168.0.0/24' | ansible.utils.ipsubnet(26, 0) }}
# → 192.168.0.0/26

{{ '10.0.0.0/24' | ansible.utils.nthhost(5) }}
# → 10.0.0.5
```

### `macaddr` / `vlan_parser`

Normalize MAC addresses and parse VLAN IDs per collection docs.

---

## Debugging & introspection filters

| Filter | Purpose |
|--------|---------|
| `type_debug` | `{{ var \| type_debug }}` → `dict`, `list`, `AnsibleUnicode`, etc. |
| `varnames` | Find variable names matching a pattern in scope |
| `comment` | Prefix lines with `#` for embedding in config templates |

**Example:**
```yaml
{{ my_list | type_debug }}
# After: "list"
```

---

## Jinja2 tests (standalone)

Use in `when:` or `{% if %}` — not only inside filters.

| Test | Example | True when |
|------|---------|-----------|
| `defined` | `my_var is defined` | Variable exists |
| `undefined` | `my_var is undefined` | Variable missing |
| `none` | `my_var is none` | Value is null |
| `failed` | `result is failed` | Registered task failed |
| `changed` | `result is changed` | Registered task changed |
| `skipped` | `result is skipped` | Task was skipped |
| `successful` | `result is successful` | Task succeeded |
| `match` | `fqdn is match('^web')` | Regex at start |
| `search` | `msg is search('error')` | Regex anywhere |
| `version` | `ansible_version is version('2.15', '>=')` | Version compare |
| `subset` | `tags is subset(['a','b'])` | Left contained in right |
| `superset` | `tags is superset(['a'])` | Right contained in left |

**Before:** `status: "DOWN"`

```yaml
when: status is search('(?i)down')
```

---

## Task & play keywords (async, delegation, control flow)

### `async` / `poll`

Run a module in the background; optionally wait with `poll`.

```yaml
- name: Long running command
  ansible.builtin.command: /usr/local/bin/long_job.sh
  async: 3600          # max runtime seconds (or omit poll to fire-and-forget)
  poll: 10             # check status every N seconds; 0 = don't wait
  register: job_result

- name: Check async status later
  ansible.builtin.async_status:
    jid: "{{ job_result.ansible_job_id }}"
  register: job_status
  until: job_status.finished
  retries: 30
  delay: 10
```

| `poll` | Behavior |
|--------|----------|
| `> 0` | Wait up to `async` seconds, polling every `poll` seconds |
| `0` | Fire and forget; use `async_status` later |
| omitted with `async` | Ansible default polling |

---

### `delegate_to` / `delegate_facts`

Run task on a **different** host than the one in inventory for this play.

```yaml
- name: Add VIP on load balancer
  ansible.builtin.command: ip addr add {{ vip }} dev eth0
  delegate_to: lb01.example.com
```

**`delegate_facts: true`** stores facts on the delegated host instead of the current inventory host.

---

### `run_once`

Run the task on the **first** host in the current batch; result can be applied to all (often with `when: inventory_hostname == ansible_play_hosts[0]` for clarity).

```yaml
- name: Create cluster token (once)
  ansible.builtin.command: create-token
  run_once: true
  register: cluster_token
```

---

### `any_errors_fatal`

Stop the play on **any** host failure (stricter than default).

```yaml
- hosts: all
  any_errors_fatal: true
```

---

### `max_fail_percentage`

Abort the play if more than N percent of hosts fail.

```yaml
- hosts: web
  max_fail_percentage: 25
```

---

### `ignore_unreachable`

Treat unreachable hosts as non-fatal for the play.

```yaml
- hosts: all
  ignore_unreachable: true
```

---

### `ignore_errors`

Continue after task failure (use sparingly).

```yaml
- name: Best-effort cleanup
  ansible.builtin.file:
    path: /tmp/maybe_missing
    state: absent
  ignore_errors: true
```

---

### `become` / `become_user` / `become_method`

```yaml
- name: Edit root-only file
  ansible.builtin.lineinfile:
    path: /etc/sudoers
    line: "%wheel ALL=(ALL) NOPASSWD: ALL"
  become: true
  become_user: root
  become_method: sudo
```

---

### `diff` / `no_log` / `check_mode`

| Keyword | Purpose |
|---------|---------|
| `diff: true` | Show file diffs on change |
| `no_log: true` | Hide task output (passwords) |
| `check_mode: true` | Dry-run (module support required) |
| `diff: false` on play | Disable diffs play-wide |

---

### `notify` / `listen` / handlers

```yaml
- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
  listen: "reload web stack"
```

Triggered by `notify: reload web stack` on a notifying task.

---

### `tags` / `skip_tags` / `meta: clear_facts`

```bash
ansible-playbook site.yml --tags deploy
ansible-playbook site.yml --skip-tags never,debug
```

```yaml
- meta: clear_facts          # refresh facts mid-play
- meta: end_play             # stop play for current host
- meta: reset_connection     # close persistent connection
```

---

## Conditionals: failed_when, changed_when, until

### `failed_when`

**Default:** module `failed` is true.

```yaml
- name: Check HTTP health
  ansible.builtin.uri:
    url: https://app/health
    status_code: 200
  register: health
  failed_when: health.status != 200 or health.json.ready != true
```

**Before (registered result):** `rc: 0` but stderr contains `ERROR`

```yaml
failed_when: result.rc != 0 or 'ERROR' in result.stderr
```

---

### `changed_when`

Suppress “changed” status (e.g. idempotent command that always returns rc 0).

```yaml
- name: Check if package installed
  ansible.builtin.command: rpm -q httpd
  register: rpm_check
  failed_when: false
  changed_when: false
```

```yaml
changed_when: "'installed' in result.stdout"
```

---

### `until` / `retries` / `delay`

Retry until condition is true.

```yaml
- name: Wait for port 443
  ansible.builtin.wait_for:
    port: 443
    timeout: 3
  register: wait_result
  until: wait_result is succeeded
  retries: 20
  delay: 5
```

| Keyword | Default | Meaning |
|---------|---------|---------|
| `retries` | 3 | Max attempts |
| `delay` | 5 | Seconds between attempts |
| `until` | — | Expression that must be true |

---

## Loops, registers, and facts

### `loop` vs `with_*`

**Preferred:** `loop` + `{{ item }}`

```yaml
- name: Install packages
  ansible.builtin.package:
    name: "{{ item }}"
    state: present
  loop:
    - httpd
    - mod_ssl
```

**Loop with dict (`dict2items`):**
```yaml
loop: "{{ users | dict2items }}"
vars:
  user: "{{ item.key }}"
  uid: "{{ item.value.uid }}"
```

### `loop_control`

```yaml
loop_control:
  label: "{{ item.name }}"    # safer play output
  index_var: idx
  pause: 2                    # seconds between iterations
  extended: true              # item.attr in nested loops
```

### `register` + `hostvars`

```yaml
- name: Get primary IP on first host
  ansible.builtin.setup:
  delegate_to: "{{ groups['web'][0] }}"
  run_once: true
  register: primary_facts

- name: Use it elsewhere
  ansible.builtin.debug:
    msg: "{{ hostvars[groups['web'][0]].ansible_default_ipv4.address }}"
```

### `set_fact` / `cacheable`

```yaml
- ansible.builtin.set_fact:
    app_version: "2.4.1"
    cacheable: true    # persist for fact cache plugins
```

---

## Execution strategy, serial, and throttle

### `serial`

Rolling updates: run play in batches.

```yaml
- hosts: web
  serial: "30%"      # or absolute: 3
  max_fail_percentage: 0
  order: sorted      # inventory order
```

**Effect:** 30% of hosts at a time; remaining hosts wait for batch to finish.

---

### `strategy`

| Strategy | Behavior |
|----------|----------|
| `linear` (default) | Each task on all hosts before next task |
| `free` | Hosts proceed independently |
| `debug` | Run with debugger on failure |
| `host_pinned` | Worker pinned to host (execution environments) |

```yaml
- hosts: all
  strategy: free
```

---

### `throttle`

Limit concurrent forks **for this task** (useful for APIs with rate limits).

```yaml
- name: Call rate-limited API
  ansible.builtin.uri:
    url: https://api.example/v1/update
  loop: "{{ items }}"
  throttle: 5
```

Only 5 hosts execute this task at once.

---

### `forks` (CLI / ansible.cfg)

Not a task keyword, but controls parallelism play-wide:

```bash
ansible-playbook site.yml -f 10
```

---

## Putting it together: real-world pattern

**Goal:** Notify only production web servers with disk > 80%.

**Before:**
```yaml
all_hosts:
  - { name: web1, group: web, env: prod, disk_pct: 85 }
  - { name: web2, group: web, env: dev,  disk_pct: 90 }
  - { name: db1,  group: db,  env: prod, disk_pct: 50 }
```

**Expression:**
```yaml
{{ all_hosts
   | selectattr('group', 'equalto', 'web')
   | selectattr('env', 'equalto', 'prod')
   | selectattr('disk_pct', 'gt', 80)
   | map(attribute='name')
   | list }}
```

**After:**
```yaml
['web1']
```

**Task:**
```yaml
- ansible.builtin.debug:
    msg: "Alert {{ item }}"
  loop: "{{ all_hosts | selectattr('group', 'equalto', 'web') | selectattr('env', 'equalto', 'prod') | selectattr('disk_pct', 'gt', 80) | map(attribute='name') | list }}"
```

---

## Sources

- [Index of all Filter Plugins](https://docs.ansible.com/ansible/latest/collections/index_filter.html)
- [selectattr filter](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/selectattr_filter.html)
- [rejectattr filter](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/rejectattr_filter.html)
- [Playbook Keywords](https://docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html)
- [Controlling playbook execution: failures and loops](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_execution.html)
- [ansible.utils IP address filters](https://docs.ansible.com/ansible/latest/collections/ansible/utils/index.html)
- [Jinja2 Template Designer Documentation](https://jinja.palletsprojects.com/en/latest/templates/)

