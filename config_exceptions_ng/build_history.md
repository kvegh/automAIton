# Build history


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
