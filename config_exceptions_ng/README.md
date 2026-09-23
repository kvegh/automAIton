# Inventory-Driven Exception Handling

This is **Configuration as Code (CaC)**. The complete configuration of
every server is defined as files in this git repository. Git is the
single source of truth.

Changes go through merge requests, are reviewed, and leave a full audit
trail in git history. No undocumented changes, no configuration drift.

**Principle: exceptions are data in git.**

All hosts run the same code. Anything host- or group-specific is expressed
as inventory data — never as host-specific playbooks or conditionals in code.

## Inventory groups

All groups are flat. A host is described by the labels it carries: a
platform, a function, a lifecycle stage, plus any network or compliance
scope. Adding a platform (say, `rhel10`) is one more label with its own
`group_vars` file; a host migrates by moving from one label to the other.

```
rhel9               (dbserver1, bastion1, webserver1, appserver2, appserver3)
rhel10              (dbserver2, sapserver1, webserver2, appserver1, appserver4)

db_servers          (dbserver1, dbserver2)
webservers          (webserver1, webserver2)
appservers          (appserver2, appserver1, appserver3, appserver4)
bastion_hosts       (bastion1)
sap                 (sapserver1)

dev                 (appserver1)
test                (appserver2)
prod                (dbserver1, bastion1, webserver1, dbserver2, sapserver1, webserver2, appserver3, appserver4)
dmz                 (bastion1, webserver1, webserver2, appserver3)
pci_scope           (sapserver1, webserver2)

sshd_upgrade_test   ()                          # promoted, now empty
sshd_hardening_test (appserver1, appserver2, webserver1)  # in progress
```

## Exception use cases

| UseCase | Type | Hosts | Implementation | Content |
|---|---|---|---|---|
| base server config | baseline | all | common role + all.yml | baseline for all RHEL servers, incl. OpenSSH 8.7p1-52 + sshd settings (promoted from sshd upgrade test) |
| rhel9 / rhel10 | platform | 5 each | group_vars | rhel_major; RHEL 10 OpenSSH build; content views derive from rhel_major |
| appservers | function | appserver2, appserver1, appserver3, appserver4 | group_vars | firewalld ports 8080/tcp, 8443/tcp |
| db_servers | function | dbserver1, dbserver2 | group_vars | sysctl tuning, THP off, mount options |
| webservers | function | webserver1, webserver2 | group_vars | firewalld ports 80/tcp, 443/tcp |
| dev | lifecycle | appserver1 | group_vars | Sat. Dev Content View + Activation Key |
| test | lifecycle | appserver2 | group_vars | Sat. Test Content View + Activation Key |
| prod | lifecycle | dbserver1, dbserver2, webserver1, webserver2, bastion1, sapserver1, appserver3, appserver4 | group_vars | Sat. Prod Content View + Activation Key |
| dmz | network | bastion1, webserver1, webserver2, appserver3 | group_vars | proxy config (env + dnf + rhsm) |
| pci_scope | compliance | sapserver1, webserver2 | group_vars + extra role | PCI log forwarding (rsyslog + retention) |
| openssh pin | host | dbserver1 | host_vars | pinned OpenSSH 8.7p1-47.el9_7 |
| sshd upgrade test | component test | — | group_vars (temporary) | done: promoted to all.yml, group left empty |
| sshd hardening test | component test | appserver1, appserver2, webserver1 | group_vars (temporary) | in progress: PasswordAuthentication no |
| additional users | host | bastion1 | host_vars | additional_user in ops group |
| sap | function | sapserver1 | group_vars + extra role | SAP packages, kernel tuning, tmpfiles |


## Directory layout

```
CODEOWNERS                      # who must approve changes to which files
site.yml                        # which groups get which roles
ansible.cfg                     # points at the inventory
collections/requirements.yml    # ansible.posix, community.general
inventory/
├── hosts.yml                   # group membership (flat)
├── group_vars/
│   ├── all.yml                 # global defaults (openssh packages, sshd settings)
│   ├── appservers.yml          # firewalld ports 8080, 8443
│   ├── bastion_hosts.yml       # SSH AllowGroups override
│   ├── db_servers.yml          # sysctl, THP, mount options
│   ├── dev.yml                 # Satellite Dev content view
│   ├── dmz.yml                 # proxy settings
│   ├── pci_scope.yml           # PCI syslog target + retention
│   ├── prod.yml                # Satellite Prod content view
│   ├── rhel9.yml               # platform: rhel_major 9
│   ├── sap.yml                 # SAP kernel tuning
│   ├── rhel10.yml              # platform: rhel_major 10, OpenSSH 9.9 build
│   ├── sshd_hardening_test.yml # component test in progress
│   ├── test.yml                # Satellite Test content view
│   └── webservers.yml          # firewalld ports 80, 443
├── host_vars/
│   ├── dbserver1.yml             # OpenSSH version pin
│   └── bastion1.yml             # additional users
roles/
├── common/                     # baseline + conditional tasks for all hosts
├── bastion_hardening/          # SSH policy + session recording
├── pci_logging/                # rsyslog forwarding + retention
└── sap_preconfigure/           # SAP packages + kernel tuning
```



## How it works

Configuration is applied in three layers:

### 1. Common role — baseline

Tasks that run on every host unconditionally. This is the standard your
RHEL servers share: package baselines, security settings, anything that
defines how you run your estate.

### 2. Common role — variable-driven tasks

The common role also contains tasks that only take effect when the right
variable is present. Ansible merges all `group_vars` and `host_vars` for
a host before any task runs. A task like:

```yaml
loop: "{{ firewalld_ports | default([]) }}"
```

opens ports on webservers and appservers (because their group_vars define
`firewalld_ports`) and does nothing on all other hosts (empty list, skipped).
The task itself is the same everywhere — only the data differs.

### 3. Extra roles — different behavior

When a host or group needs different software or services (not just
different values), it gets its own role. The role is assigned through
group membership in the inventory and a play in `site.yml`.

To add an existing extra role to a new host: add the host to the group
in `hosts.yml`. That's it.

To introduce an entirely new behavior: create the role, add a group to
the inventory, and add a play to `site.yml`.

### What does host X get?

Look at three things:

1. **Common role** — always runs.
2. **Group memberships** in `hosts.yml` — determines which extra roles
   and which group_vars apply.
3. **`host_vars/X.yml`** — if it exists, that file lists everything
   that makes this specific host different.

### If you know Hiera

The inventory is Ansible's data layer, the way Hiera is Puppet's. The
files map almost one to one:

| Hiera | Ansible |
|---|---|
| `data/nodes/<certname>.yaml` | `host_vars/<host>.yml` |
| `data/roles/<role>.yaml` | `group_vars/<group>.yml` |
| `data/stage/<stage>.yaml` | `group_vars/<stage>.yml` |
| `data/common.yaml` | `group_vars/all.yml` |
| hierarchy in `hiera.yaml` | fixed: `host_vars` > `group_vars` > `all` |

One difference: in Hiera you define the hierarchy yourself. In Ansible
the order is fixed — a host value beats a group value beats the default.
Anything more specific is done with group nesting, not with configuration.

## Adding a new exception

- **Different value, multiple hosts** → create a group, add a
  `group_vars/<group>.yml` file. If the common role already has a
  matching task, you're done. If not, add a variable-driven task.
- **Different value, single host** → add a `host_vars/<host>.yml` file.
- **Different behavior** → create a role, add the group to the inventory,
  add a play to `site.yml`.
- **Two or more hosts with the same `host_vars`** → that is a group. Create
  it, move the values to `group_vars`, delete the `host_vars` files.
  `db_servers` (dbserver1, dbserver2) is the example: shared sysctl and mount
  options live in one `group_vars` file, not in two identical `host_vars`.
- **No exception → no file.** webserver1 and webserver2 have no `host_vars` at
  all; their entire configuration comes from group membership.

## Testing a new component version

A new version of a component (here: sshd upgrade) is tested on a few hosts
before it becomes the baseline. No branch, no separate inventory.

**Test:**

1. Create a group `sshd_upgrade_test` in `hosts.yml` with one host per stage
   (dev, test, prod).
2. Put the new version and its settings in `group_vars/sshd_upgrade_test.yml`.
3. Run `site.yml` — or just `--limit sshd_upgrade_test`.

The test hosts stay in all their other groups. They keep receiving every
baseline change during the test. Everything else in the inventory is
untouched.

**Promote:**

1. Move the values from `group_vars/sshd_upgrade_test.yml` into `all.yml`.
2. Delete `group_vars/sshd_upgrade_test.yml` and the group in `hosts.yml`.

The inventory is back to where it started, and the new version is the
baseline for all hosts. Host-level exceptions (like the pin on dbserver1)
are unaffected — `host_vars` still wins.

On `main` right now: `sshd_upgrade_test` went through both steps and is
empty; `sshd_hardening_test` is in the test step.

## Who may change what

One branch does not mean everyone may change everything. Three separate
questions, three separate controls:

- **Who may see** — decided by the repository. Read access on git
  platforms is always repository-wide; a branch never hides anything
  from someone who can read the repository.
- **Who may change** — decided by the protected branch and `CODEOWNERS`.
- **Who may run against which hosts** — decided in AAP, by job template
  and inventory permissions.

`main` is protected: no direct pushes, changes only through merge
requests. Who must approve a merge request depends on which files it
touches — that is what `CODEOWNERS` defines:

- **Platform team** owns the baseline: `roles/`, `site.yml`, `all.yml`,
  and `hosts.yml`. Adding a host to a test group is a `hosts.yml` change,
  so which hosts a test can reach is always reviewed by the platform team.
- **Component team** owns only its own test group file,
  `group_vars/sshd_hardening_test.yml`. Inside that file they are free; outside
  it they need the platform team's approval.

The settings that make this enforced, and the matching rules, are in the
header of `CODEOWNERS`. This is GitLab syntax — GitHub reads the same
file name but not the `[Section]` syntax.

## Converting an existing branch

A long-lived feature branch becomes inventory data in three steps:

1. Diff the branch against `main`.
2. Classify every difference: different value → `group_vars`;
   different behavior → role; no longer needed → delete.
3. Close the branch.

The third category is real. Example, found in a branch last touched in
2021 and serving an integration that had since been decommissioned:

```yaml
ssl_ciphers: "DES-CBC3-SHA:RC4"
ssl_min_protocol: "TLSv1"
```

Nothing was migrated and no group was created; the branch was closed.
Converting a branch is also the moment to remove what no longer belongs.

## Build history

Each feature was added in a separate commit. You can compare any two
tags on GitHub to see exactly what changed:

| Tag | Description | Diff |
|---|---|---|
| `v0-skeleton` | empty project scaffold | — |
| `v1-inventory` | inventory with all groups | [v0-skeleton...v1-inventory](https://github.com/kvegh/automAIton/compare/v0-skeleton...v1-inventory) |
| `v2-bastion` | bastion hardening role | [v1...v2](https://github.com/kvegh/automAIton/compare/v1-inventory...v2-bastion) |
| `v3-appservers` | appserver firewalld ports | [v2...v3](https://github.com/kvegh/automAIton/compare/v2-bastion...v3-appservers) |
| `v4-dbservers` | db_servers sysctl + mount options | [v3...v4](https://github.com/kvegh/automAIton/compare/v3-appservers...v4-dbservers) |
| `v5-webservers` | webserver firewalld ports | [v4...v5](https://github.com/kvegh/automAIton/compare/v4-dbservers...v5-webservers) |
| `v6-readme-layers` | README: three config layers | [v5...v6](https://github.com/kvegh/automAIton/compare/v5-webservers...v6-readme-layers) |
| `v7-lifecycle` | dev/test/prod satellite registration | [v6...v7](https://github.com/kvegh/automAIton/compare/v6-readme-layers...v7-lifecycle) |
| `v8-network-compliance` | DMZ proxy + PCI log forwarding | [v7...v8](https://github.com/kvegh/automAIton/compare/v7-lifecycle...v8-network-compliance) |
| `v9-sniper` | host-level: openssh pin, users, SAP | [v8...v9](https://github.com/kvegh/automAIton/compare/v8-network-compliance...v9-sniper) |
| `v10-readme` | README: full documentation | [v9...v10](https://github.com/kvegh/automAIton/compare/v9-sniper...v10-readme) |
| `v11-cac-intro` | README: CaC + single source of truth intro | [v10...v11](https://github.com/kvegh/automAIton/compare/v10-readme...v11-cac-intro) |
| `v12-component-test` | sshd upgrade test group, one host per stage | [v11...v12](https://github.com/kvegh/automAIton/compare/v11-cac-intro...v12-component-test) |
| `v13-promote` | sshd upgrade promoted to all.yml, test group dissolved | [v12...v13](https://github.com/kvegh/automAIton/compare/v12-component-test...v13-promote) |
| `v14-codeowners` | CODEOWNERS: path-level approval on one main | [v13...v14](https://github.com/kvegh/automAIton/compare/v13-promote...v14-codeowners) |
| `v15-consolidate` | README: branch conversion, group + footprint rules; old `config_exceptions` removed | [v14...v15](https://github.com/kvegh/automAIton/compare/v14-codeowners...v15-consolidate) |
| `v16-puppet-bridge` | README: Hiera mapping, see/change/run, enforcement by schedule | [v15...v16](https://github.com/kvegh/automAIton/compare/v15-consolidate...v16-puppet-bridge) |

## Run

```
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml                   # everything
ansible-playbook site.yml --tags openssh    # just OpenSSH tasks
ansible-playbook site.yml --tags firewall   # just firewalld tasks
ansible-playbook site.yml --tags sap        # just SAP hosts
ansible-playbook site.yml --tags pci        # just PCI scope
```

Ansible enforces when a job runs; there is no resident agent that
re-applies the configuration every 30 minutes the way a Puppet agent
does. For continuous enforcement, schedule the baseline job template
in AAP — or trigger it from Event-Driven Ansible.
