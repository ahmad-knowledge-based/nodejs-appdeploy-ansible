# AGENTS.md

## Purpose
This repository manages Node.js application deployment with Ansible.
It provisions prerequisite packages, installs Node.js via NVM, manages PM2, configures Nginx, and deploys two app types:
- PM2-backed applications from `apps`
- static Vite applications from `vite_apps`

## Session Bootstrap
When starting a fresh session, rebuild context from the repository instead of assuming prior chat history.
Read these files first:
- `README.md`
- `ansible.cfg`
- `playbooks/site.yml`
- `inventories/deploy/hosts.ini`
- `inventories/deploy/group_vars/all.yml`
- `inventories/deploy/group_vars/dev.yml`
- `inventories/deploy/group_vars/prod.yml`

Then inspect these roles before editing behavior:
- `roles/base_user`
- `roles/nodejs`
- `roles/pm2`
- `roles/nginx`
- `roles/letsencrypt`
- `roles/app_deploy`
- `roles/vite_app_deploy`

If the available context window is tight, summarize findings locally and keep only the active role plus the relevant inventory vars in working context.

## Main Execution Model
Primary entry point:
- `playbooks/site.yml`

Execution flow:
1. `base_user`
2. `nodejs`
3. `pm2`
4. `nginx`
5. `letsencrypt`
6. loop over `apps` with `app_deploy`
7. loop over `vite_apps` with `vite_app_deploy`

Important tags:
- `prereq`
- `letsencrypt`
- `deploy`
- `app_code`
- `app_pm2`
- `app_nginx`
- `vite_deploy`
- `vite_code`
- `vite_nginx`

Use positive tag selection for deploy sub-phases. Do not use `--skip-tags` to
omit sub-phases from looped deploy roles because the include task carries the
sub-phase tags and may skip the whole include.

Preserve tag semantics when editing tasks.

## Variable Source Of Truth
Shared variables live in:
- `inventories/deploy/group_vars/all.yml`

Treat these as the primary shared variables:
- `developer_user`
- `developer_home`
- `github_username`
- `github_token`
- `github_persist_auth_in_remote`
- `nvm_version`
- `node_versions`
- `node_default_version`
- `nginx_proxy_buffers`
- `nginx_proxy_buffer_size`
- `letsencrypt_email`
- `letsencrypt_staging`

Avoid reintroducing duplicated literal defaults across multiple roles.
If a role needs compatibility aliases, point them back to the shared global variables instead of hardcoding values again.

## Inventory Model
Environment-specific app definitions live in:
- `inventories/deploy/group_vars/dev.yml`
- `inventories/deploy/group_vars/prod.yml`

Expected shape for `apps` items:
- `name`
- `domain`
- `port`
- `repo`
- `branch`
- `app_dir`
- `env_src`
- `needs_install`
- `needs_build`
- `pm2_mode`
- optional `pm2_script`
- optional `pm2_args`
- optional `node_version`
- optional `ssl_fullchain`
- optional `ssl_privkey`

Expected shape for `vite_apps` items:
- `name`
- `domain`
- `repo`
- `branch`
- `app_dir`
- `static_root`
- `needs_install`
- `needs_build`
- optional `npm_install_cmd`
- optional `node_version`
- optional `ssl_fullchain`
- optional `ssl_privkey`

Do not casually rename `app` or `vite_app` loop variables because the role tasks depend on them.

## Role Ownership
Role responsibilities are intentionally split:
- `base_user`: create deploy user and home directory
- `nodejs`: install NVM and Node versions
- `pm2`: install PM2 and systemd service
- `nginx`: install base nginx and shared include config only
- `letsencrypt`: install certbot, create temporary HTTP ACME challenge vhosts, and issue certificates
- `app_deploy`: clone app repo, deploy `.env`, install/build, run PM2 app, write PM2 app vhost
- `vite_app_deploy`: clone vite repo, install/build, write static site vhost

Do not move final app-facing vhost ownership into `roles/nginx` unless there is a deliberate architectural change.
Current active vhost templates are split by scheme:
- `roles/app_deploy/templates/vhost.conf-http.j2`
- `roles/app_deploy/templates/vhost.conf-https.j2`
- `roles/vite_app_deploy/templates/vhost.conf-http.j2`
- `roles/vite_app_deploy/templates/vhost.conf-https.j2`

`roles/nginx` currently owns only base nginx setup and shared include files.

## LetsEncrypt
Automated LetsEncrypt provisioning is implemented in `roles/letsencrypt` (tag `letsencrypt`).
Ownership split:
- `nginx`: base package install and shared includes
- `letsencrypt`: certbot install, temporary HTTP ACME challenge vhost, certificate issuance
- `app_deploy` and `vite_app_deploy`: final HTTPS vhost rendering

`letsencrypt_domains` is auto-extracted from the `domain` fields of `apps` and `vite_apps`.
Avoid letting both `nginx` and app deploy roles become competing owners of the same final site config.

## Secrets And Auth
The playbook loads:
- `vault/github_token.yml`

Current `.gitignore` behavior:
- `vault/github_token.yml` is ignored and should remain local; keep `vault/github_token.example.yml` as the public placeholder.
- `vault/dev/*` and `vault/prod/*` are ignored, including environment files referenced by `env_src`.
- `vault/dev/.gitkeep` and `vault/prod/.gitkeep` are tracked only to preserve directory shape.
- Treat app `.env` vault files as local/operator-provided inputs unless there is an explicit decision to track sanitized examples.

Current Git auth strategy:
- clone and update repositories using authenticated HTTPS remote URLs
- always clone using the authenticated remote URL
- `github_persist_auth_in_remote` controls whether `origin` stays authenticated or is reset back to the clean repository URL
- do not add `.netrc`-based auth back into the repo unless there is an explicit architecture change

When editing Git auth flow:
- preserve `no_log: true` around secret-bearing tasks
- preserve the authenticated-clone strategy
- only reset `origin` to the clean URL when `github_persist_auth_in_remote` is false

Never print tokens in logs, debug output, or comments.

## Editing Guidance
Prefer small, behavior-preserving changes.
When editing shell tasks:
- preserve `set -euo pipefail`
- preserve the NVM environment bootstrap unless intentionally refactoring it
- improve idempotency when practical

When editing app deploy roles:
- keep repo auth flow consistent
- preserve `become_user`
- preserve per-app `node_version` override support
- avoid introducing a second source of truth for developer paths or Node version

When editing inventory vars:
- prefer changing shared values in `group_vars/all.yml`
- use `dev.yml` and `prod.yml` only for environment-specific app definitions
- if adding shared Let's Encrypt defaults, keep them in `group_vars/all.yml` unless they are environment-specific

## Validation Guidance
Before finishing changes, perform lightweight verification when possible:
- inspect referenced vars with `rg`
- check YAML validity
- run Ansible syntax checks or dry runs if the environment allows it

Useful commands:
- `ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags prereq --limit "prod:dev"`
- `ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags letsencrypt --limit prod`
- `ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags app_code,app_pm2 --limit dev`
- `ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --check`
- `ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --syntax-check`

If local Ansible execution is blocked by missing vault password files or sandbox temp directory issues, state that explicitly in the handoff.

## Known Sharp Edges
Watch for these before making broader refactors:
- multiple deploy tasks still use `changed_when: true`, which can reduce idempotency signal quality
- deploy clone tasks still use `force: true`, which may be too destructive for some environments
- authenticated remote URLs may remain in the repo remote config on the server when `github_persist_auth_in_remote` is true
- role defaults for app deploy compatibility aliases still contain literal fallback values; prefer shared inventory values for active configuration
- `README.md`, `AGENTS.md`, and `.gitignore` should be kept in sync with the intended `vault/` tracking policy

## Handoff Notes For Future Sessions
When ending a session after substantial analysis or refactoring:
- leave the repository in a readable state
- update this file if architectural guidance changed
- mention any unresolved risks in the final response

When starting a new session:
- trust repository state over prior conversational memory
- re-read the current role/task files before assuming an earlier plan still applies
