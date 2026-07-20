# Inventory-Driven Exception Handling

**Principle: exceptions are data in git.**

All hosts run the same code. Anything host- or group-specific is expressed
as inventory data — never as host-specific playbooks or conditionals in code.

## Exception Use Cases

| UseCase | Type | Hosts | Implementation | Content |
|---|---|---|---|---|
| base server config | baseline | all | common role | baseline for all RHEL servers |
| appservers | function | server4, server8 | group_vars | firewalld ports 8080/tcp, 8443/tcp |

## Rules

1. **Same behavior, different values** → `group_vars` / `host_vars`.
2. **Different behavior** → a role, selected by group membership. No hostnames in playbooks.
3. **Defaults have zero footprint.** A default host has no `host_vars` file
   and appears in no exception group.

## Run

```
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml
```
