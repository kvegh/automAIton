# Edge cases

## Overlapping group variables

A host belongs to several groups at once. When two of those groups set the
**same variable**, they sit at the same precedence level and Ansible does
**not** merge them — it picks one group's value (by `ansible_group_priority`,
then, as a last resort, group name). Relying on that is fragile: a value
wins by accident, and adding a group later can silently change the outcome.

Example: `webserver1` is in `webservers` (opens 80/443) and in `prod` (which
also wants 9100 for monitoring). If both define `firewalld_ports`, one list
silently replaces the other.

There are two correct answers, depending on what you actually mean.

### You want to combine them (usual case)

The two groups describe two different axes — a function *and* a stage — so
they are two variables, not one. Give each its own key and union them in the
task:

```yaml
# group_vars/webservers.yml
firewalld_ports_function:
  - 80/tcp
  - 443/tcp

# group_vars/prod.yml
firewalld_ports_stage:
  - 9100/tcp
```

```yaml
# roles/common/tasks/main.yml
- name: Open firewalld ports
  ansible.posix.firewalld:
    port: "{{ item }}"
    permanent: true
    immediate: true
    state: enabled
  loop: >-
    {{ (firewalld_ports_function | default([]))
     + (firewalld_ports_stage    | default([])) }}
  tags: [firewall]
```

A prod webserver gets all three ports; a test webserver gets 80/443 plus
whatever `test` adds. No shared key, no precedence fight, and the behaviour
is visible where the variable is used.

### You want one group to override the other

Sometimes the intent really is "this group's value replaces the others" —
a `security_lockdown` or `compliance` group whose settings must win over
whatever a function or stage set. Make that explicit with
`ansible_group_priority` on the authoritative group:

```yaml
# group_vars/security_lockdown.yml
ansible_group_priority: 10      # default is 1; highest wins
sshd_settings:
  MaxAuthTries: 2
  PasswordAuthentication: "no"
```

Among same-level groups defining the same variable, the highest priority is
evaluated last and wins. This replaces the group-name tiebreak with a
deterministic, on-purpose choice.

**Use it sparingly.** `ansible_group_priority` is action at a distance: the
override is not visible where the variable is used, only by knowing each
group's priority number — the same "hold the hierarchy in your head" problem
that the flat model set out to avoid. Reserve it for a small number of
clearly authoritative groups, and always comment why the number is there.
For anything additive, use the union pattern above instead.

**Rule of thumb:** the same key in two groups a host shares means you have
either two axes under one name (split the key, union in the task) or one
authoritative axis (set `ansible_group_priority` explicitly). Never leave it
to the group-name tiebreak.
