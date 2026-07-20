# Inventory-Driven Configuration Pattern

**Principle: exceptions are data in git.**

All hosts run the same code. Anything host-specific is expressed as
inventory data — never as host-specific playbooks or conditionals in code.

## Layout

```
site.yml                        # orchestration: which groups get which roles
ansible.cfg                     # points at the inventory; run: ansible-playbook site.yml
collections/requirements.yml    # ansible.posix, community.general
inventory/
├── hosts.yml                   # group membership
├── group_vars/
│   ├── all.yml                 # global defaults
│   ├── db_servers.yml          # sysctl tuning + mount options (cluster)
│   ├── dmz.yml                 # proxy settings
│   ├── dev.yml                 # Satellite lifecycle: Development
│   ├── test.yml                # Satellite lifecycle: Test
│   ├── prod.yml                # Satellite lifecycle: Production
│   ├── pci_scope.yml           # PCI log-forwarding parameters
│   ├── webservers.yml          # firewalld ports: 80, 443
│   ├── appservers.yml          # firewalld ports: 8080, 8443
│   └── sap.yml                 # SAP kernel tuning
└── host_vars/
    ├── server1.yml             # OpenSSH version pin
    └── server2.yml             # bastion parameters
roles/
├── common/                     # baseline for every host (value-driven tasks)
├── bastion_hardening/          # bastion SSH policy + session recording
├── pci_logging/                # PCI DSS rsyslog forwarding + retention
└── sap_preconfigure/           # SAP prerequisite packages + kernel tuning
```

## Exception categories demonstrated

| # | Exception | Category | Location |
|---|---|---|---|
| 1 | sysctl: swappiness, hugepages, THP off | value (cluster) | `group_vars/db_servers.yml` |
| 2 | Proxy (env + dnf + rhsm) | value + common role | `group_vars/dmz.yml` |
| 3 | Satellite content view / activation key | value | `group_vars/{dev,test,prod}.yml` |
| 4 | Mount options (noatime) | value (cluster) | `group_vars/db_servers.yml` |
| 5 | PCI log forwarding | behavior | `pci_logging` role + `pci_scope` group |
| 6 | Firewalld ports per app | value list + common role | `group_vars/{webservers,appservers}.yml` |
| 7 | SAP prerequisites | behavior | `sap_preconfigure` role + `sap` group |
| 8 | Old TLS cipher override | drift / dead code | deleted — see below |

UC1 + UC4 cluster on the same `db_servers` group, showing the pattern:
when multiple hosts share the same exceptions, promote from `host_vars` to
a dedicated group — exceptions become groups, not snowflakes.

## Rules

1. **Same behavior, different values → variables** (`group_vars`/`host_vars`).
   Example: `host_vars/server1.yml` pins an exact OpenSSH build.
2. **Different behavior → a role, selected by group membership** in the
   inventory. Example: hosts in `bastion_hosts` additionally get the
   `bastion_hardening` role. No hostnames in playbooks, ever.
3. **Defaults have zero footprint.** A host with no `host_vars` file
   inherits everything from its groups. `server3`–`server5` are examples:
   their entire configuration comes from group membership alone.

The question "what is different about host X?" is always answered by one
file: `inventory/host_vars/X.yml` — plus its group memberships in
`hosts.yml`. Auditable, diffable, reviewable in a merge request.

## Migration archaeology: what we deleted (UC8)

During branch archaeology (Step 2 of the migration), the following was
found in an old feature branch and classified as **drift / dead code**:

```yaml
# Found in branch feature/legacy-tls-compat (last commit 2021-09-14)
# Forced weak TLS ciphers for a since-decommissioned legacy integration.
ssl_ciphers: "DES-CBC3-SHA:RC4"
ssl_min_protocol: "TLSv1"
```

No inventory entry was created — the correct action was deletion, not
migration. The integration it served no longer exists; keeping it would
have been a security liability. This is the hidden value of branch
archaeology: after years of long-lived branches, expect 30–50% of diffs
to be dead legacy nobody can explain. The migration cleans the estate.

## Run

```
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml                   # everything
ansible-playbook site.yml --tags openssh    # partial run
ansible-playbook site.yml --tags sap        # SAP hosts only
ansible-playbook site.yml --tags pci        # PCI scope only
```
