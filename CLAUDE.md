# CLAUDE.md

Operating guidance for this Ansible Node.js deployment repo.
The full architecture/handoff notes live in @AGENTS.md — read it first.

## What this repo is
Ansible automation that provisions a host (user, Node via NVM, PM2, Nginx,
Let's Encrypt) and deploys two app types from inventory:
- `apps`        → PM2-backed Node services (roles/app_deploy)
- `vite_apps`   → static Vite SPAs       (roles/vite_app_deploy)
Entry point: playbooks/site.yml.

## Hard rules (do not violate without being asked)
- Use FQCN module names everywhere (`ansible.builtin.*`). No short names.
- Keep `no_log: true` on every task that touches `github_token` or `.env`.
  Never echo a token in tasks, debug, comments, or commit messages.
- In shell tasks, preserve `set -euo pipefail` and the NVM bootstrap
  (`source $NVM_DIR/nvm.sh && nvm use ...`) — npm/node/pm2 break without it.
- Preserve `become_user` and per-app `node_version` overrides in deploy roles.
- Don't rename the `app` / `vite_app` / `letsencrypt_domain` loop vars.
- Preserve tag semantics: prereq / letsencrypt / deploy / vite_deploy.

## Variable source of truth
- Shared defaults: inventories/deploy/group_vars/all.yml — change them HERE.
- Per-env app lists: dev.yml / prod.yml only.
- Role defaults are aliases back to the global vars
  (e.g. `app_deploy_developer_user: "{{ developer_user | default('developer') }}"`).
  Never hardcode a second literal default for developer paths or Node version.

## Role ownership (don't blur these)
- nginx        → base install + shared includes only.
- letsencrypt  → certbot, temp ACME challenge vhost, cert issuance.
- app_deploy / vite_app_deploy → own the FINAL site vhosts. Keep it that way.

## Secrets
- vault/github_token.yml is local/ignored, encrypted with Ansible Vault, and loaded by the playbook.
- vault/github_token.example.yml is the public placeholder.
- Env files referenced via `env_src` (e.g. vault/dev/<app>.env), deployed mode 0600.
- Vault password file: ~/.config/ansible/vault_pass.txt (never commit).

## Validate before finishing
- `ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --syntax-check`
- `ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --check`
- `rg` to confirm any var you touched is consistent across roles + inventory.
- If vault password / sandbox temp dirs block a run, say so explicitly.

## Known sharp edges
- Several deploy tasks use `changed_when: true` → weak idempotency signal.
- Clone tasks use `force: true` → can be destructive on the server.
- Authenticated remote URL may persist in server git config when
  `github_persist_auth_in_remote: true`.
