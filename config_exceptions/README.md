# Inventory-Driven Configuration Pattern

**Principle: exceptions are data in git.**

All hosts run the same code. Anything host-specific is expressed as
inventory data — never as host-specific playbooks or conditionals in code.

## Layout

```
site.yml                    # orchestration: which groups get which roles
ansible.cfg                 # points at the inventory; run: ansible-playbook site.yml
inventory/
├── hosts.yml               # group membership
├── group_vars/all.yml      # global defaults
└── host_vars/<host>.yml    # per-host exceptions, one file per host
roles/
├── common/                 # baseline for every host
└── bastion_hardening/      # additional behavior, applied via group membership
```

## Rules

1. **Same behavior, different values → variables** (`group_vars`/`host_vars`).
   Example: `host_vars/server1.yml` pins an exact OpenSSH build.
2. **Different behavior → a role, selected by group membership** in the
   inventory. Example: hosts in `bastion_hosts` additionally get the
   `bastion_hardening` role. No hostnames in playbooks, ever.
3. **Defaults have zero footprint.** A default host has no `host_vars` file
   and appears in no exception group. `server3`/`server4` are examples.

The question "what is different about host X?" is always answered by one
file: `inventory/host_vars/X.yml` — plus its group memberships in
`hosts.yml`. Auditable, diffable, reviewable in a merge request.

## Run

```
ansible-playbook site.yml                 # everything
ansible-playbook site.yml --tags openssh  # partial run
```
