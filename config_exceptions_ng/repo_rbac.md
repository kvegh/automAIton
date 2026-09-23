# Repository RBAC — who owns what

When teams contribute to a shared automation repository, split ownership by
what each file controls, not by team boundary. Code decentralizes to the
teams; activation and the shared baseline stay with the platform team.

| What | Where | Who |
|---|---|---|
| Role code + sane defaults | the team's collection (`defaults/main.yml`) | team, fully — versioned in their repo |
| Site-specific values | `group_vars/<their group>.yml` | team, via CODEOWNERS on that file |
| Version pin of the collection | `requirements.yml` | platform (team proposes bump) |
| New group / membership | `hosts.yml` | platform |
| group → role binding | `site.yml` | platform |

**Why this split:**

- A team ships and versions its own **roles** as a collection, and changes
  its own **values** in its own `group_vars` file, without touching the
  center — so day-to-day work needs no platform involvement.
- Turning a role *on* for hosts (`site.yml`) and deciding *which* hosts a
  change reaches (`hosts.yml`) stay with the platform team, because that is
  the moment a new behavior starts running on real machines — the
  blast-radius decision. It is rare (once per new role) and worth a review.
- `host_vars` (single-host exceptions) stays platform-gated even where
  `group_vars` is delegated: it is the highest precedence, so it can
  override the baseline — including a security control — for that one host.
  Teams propose a pin via merge request; platform approves it.
- The baseline `all.yml` is platform-only and never assembled from team
  fragments, so no team can change an estate-wide value.

Without CODEOWNERS (GitLab Free), path-level delegation is not available:
`main` is protected, everyone opens merge requests, and only the platform
group can merge. Per-team repositories (each team a Maintainer of its own,
assembled via collections) are the free-tier way to regain team autonomy.
