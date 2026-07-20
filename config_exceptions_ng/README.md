# Inventory-Driven Exception Handling

**Principle: exceptions are data in git.**

All hosts run the same code. Anything host- or group-specific is expressed
as inventory data — never as host-specific playbooks or conditionals in code.

## Directory layout

```
site.yml                        # which groups get which roles
ansible.cfg                     # points at the inventory
collections/requirements.yml    # ansible.posix, community.general
inventory/
├── hosts.yml                   # group membership (cascaded)
├── group_vars/
│   ├── all.yml                 # global defaults (openssh packages)
│   ├── appservers.yml          # firewalld ports 8080, 8443
│   ├── bastion_hosts.yml       # SSH AllowGroups override
│   ├── db_servers.yml          # sysctl, THP, mount options
│   ├── dev.yml                 # Satellite Dev content view
│   ├── dmz.yml                 # proxy settings
│   ├── pci_scope.yml           # PCI syslog target + retention
│   ├── prod.yml                # Satellite Prod content view
│   ├── test.yml                # Satellite Test content view
│   └── webservers.yml          # firewalld ports 80, 443
├── host_vars/
│   ├── server1.yml             # OpenSSH version pin
│   ├── server2.yml             # additional users
│   └── server6.yml             # SAP kernel tuning
roles/
├── common/                     # baseline + conditional tasks for all hosts
├── bastion_hardening/          # SSH policy + session recording
├── pci_logging/                # rsyslog forwarding + retention
└── sap_preconfigure/           # SAP packages + kernel tuning
```

## Inventory groups

The inventory uses cascaded groups. Function groups are children of `rhel9`,
so each host is defined exactly once. Lifecycle, network, and compliance
groups cut across functions and stay flat.

```
rhel9
├── db_servers      (server1, server5)
├── webservers      (server3, server7)
├── appservers      (server4, server8)
├── bastion_hosts   (server2)
└── sap             (server6)

dev                 (server8)
test                (server4)
prod                (server1, server2, server3, server5, server6, server7)
dmz                 (server2, server3, server7)
pci_scope           (server6, server7)
```

## Exception use cases

| UseCase | Type | Hosts | Implementation | Content |
|---|---|---|---|---|
| base server config | baseline | all | common role | baseline for all RHEL servers |
| appservers | function | server4, server8 | group_vars | firewalld ports 8080/tcp, 8443/tcp |
| db_servers | function | server1, server5 | group_vars | sysctl tuning, THP off, mount options |
| webservers | function | server3, server7 | group_vars | firewalld ports 80/tcp, 443/tcp |
| dev | lifecycle | server8 | group_vars | Sat. Dev Content View + Activation Key |
| test | lifecycle | server4 | group_vars | Sat. Test Content View + Activation Key |
| prod | lifecycle | server1,2,3,5,6,7 | group_vars | Sat. Prod Content View + Activation Key |
| dmz | network | server2, server3, server7 | group_vars | proxy config (env + dnf + rhsm) |
| pci_scope | compliance | server6, server7 | group_vars + extra role | PCI log forwarding (rsyslog + retention) |
| openssh pin | host | server1 | host_vars | pinned OpenSSH 8.7p1-47.el9_7 |
| additional users | host | server2 | host_vars | additional_user in ops group |
| SAP prerequisites | host | server6 | host_vars + extra role | SAP packages, kernel tuning, tmpfiles |

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

## Adding a new exception

- **Different value, multiple hosts** → create a group, add a
  `group_vars/<group>.yml` file. If the common role already has a
  matching task, you're done. If not, add a variable-driven task.
- **Different value, single host** → add a `host_vars/<host>.yml` file.
- **Different behavior** → create a role, add the group to the inventory,
  add a play to `site.yml`.

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

## Run

```
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml                   # everything
ansible-playbook site.yml --tags openssh    # just OpenSSH tasks
ansible-playbook site.yml --tags firewall   # just firewalld tasks
ansible-playbook site.yml --tags sap        # just SAP hosts
ansible-playbook site.yml --tags pci        # just PCI scope
```
