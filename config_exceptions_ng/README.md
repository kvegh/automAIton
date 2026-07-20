# Inventory-Driven Exception Handling

**Principle: exceptions are data in git.**

All hosts run the same code. Anything host- or group-specific is expressed
as inventory data — never as host-specific playbooks or conditionals in code.

## Exception Use Cases

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

### 1. Common role — unconditional baseline

Tasks that every host gets, no questions asked. Fixed security policy,
package standards, anything that defines "this is how we run RHEL here."

### 2. Common role — conditional tasks

Tasks that exist in the common role but only execute when the matching
variable is present. For example, the firewalld task loops over
`firewalld_ports | default([])` — if a host's group defines that variable,
the ports open. If not, the task is silently skipped. No error, no action.

The variables come from `group_vars/` and `host_vars/`. Ansible merges
all applicable variables for a host before any task runs. The tasks never
need to know *where* a variable came from — they only ask "is it there?"

### 3. Extra roles — structural behavior differences

When a group needs fundamentally different software or services (not just
different values), it gets its own role. The role is assigned via group
membership in the inventory and a dedicated play in `site.yml`.

Example: `bastion_hosts` group → `bastion_hardening` role. Adding a host
to the group is enough — no code change, no new play.

## Rules

1. **Same behavior, different values** → `group_vars` / `host_vars`.
2. **Different behavior** → a role, selected by group membership. No hostnames in playbooks.
3. **Defaults have zero footprint.** A default host has no `host_vars` file
   and appears in no exception group.

The answer to "what does host X get?" is always:
common role (always) + extra roles from its group memberships + variables
from its `group_vars` and `host_vars`.

## Run

```
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml
```
