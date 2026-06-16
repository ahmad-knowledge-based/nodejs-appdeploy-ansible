# Node.js App Deployment with Ansible

This repository provisions and deploys Node.js applications with Ansible.
It currently supports:

- base server setup
- Node.js installation through NVM
- PM2 process management for backend or SSR-style apps
- Nginx reverse proxy and static hosting
- optional Let's Encrypt provisioning through a dedicated role

## Repository Scope

This repo is used as an Ansible deployment workspace.
Inventory, domains, repositories, and vault-backed secrets are expected to be customized per environment.

## Architecture Map

Main entry point:

- `playbooks/site.yml`

Execution flow:

1. `base_user`
2. `nodejs`
3. `pm2`
4. `nginx`
5. `letsencrypt`
6. loop `apps` through `app_deploy`
7. loop `vite_apps` through `vite_app_deploy`

Role responsibilities:

- `roles/base_user`: create the deploy user and home directory
- `roles/nodejs`: install NVM and the configured Node versions
- `roles/pm2`: install PM2 and its systemd service
- `roles/nginx`: install nginx and shared include config
- `roles/letsencrypt`: issue certificates with `certbot` using temporary ACME nginx vhosts
- `roles/app_deploy`: clone app repos, deploy `.env`, install/build, manage PM2 apps, write app nginx vhosts
- `roles/vite_app_deploy`: clone vite repos, install/build, and write static nginx vhosts

Current tag layout:

- `prereq`
- `letsencrypt`
- `deploy` — full PM2 app deploy (code + PM2 + nginx vhost)
- `vite_deploy` — full Vite app deploy (code + nginx vhost)

The `deploy` and `vite_deploy` roles are further split into sub-phase tags so a
deployment can be scoped to a single stage:

- `app_code` — clone, `.env`, install, build (PM2 apps)
- `app_pm2` — recreate/save the PM2 process (PM2 apps)
- `app_nginx` — render + enable the app nginx vhost (PM2 apps)
- `vite_code` — clone, `.env`, install, build (Vite apps)
- `vite_nginx` — render + enable the static nginx vhost (Vite apps)

Running `--tags deploy` (or `--tags vite_deploy`) still executes every stage, so
existing behavior is unchanged. The sub-tags let you stop at, or isolate, a
stage — **always select stages with `--tags` (positive selection), not
`--skip-tags`** (see the warning below):

| Goal | Command |
| --- | --- |
| Full PM2 app deploy | `--tags deploy` |
| Deploy code + PM2, no nginx | `--tags app_code,app_pm2` |
| Pull code + rebuild only | `--tags app_code` |
| Recreate the PM2 process only | `--tags app_pm2` |
| Refresh the app nginx vhost only | `--tags app_nginx` |
| Full Vite app deploy | `--tags vite_deploy` |
| Rebuild Vite without touching nginx | `--tags vite_code` |
| Refresh the Vite nginx vhost only | `--tags vite_nginx` |

> **Do not use `--skip-tags` with the sub-phase tags.** The deploy roles are
> pulled in via a looped `include_role`, and every sub-tag is listed on that
> include so positive `--tags` selection can reach it. As a side effect the
> include task itself carries all the sub-tags, so `--skip-tags app_nginx`
> skips the *entire* include (the whole loop) — not just the nginx step — and
> nothing deploys. Use positive selection (e.g. `--tags app_code,app_pm2`) to
> "stop before nginx" instead.

## Directory Layout

- `playbooks/site.yml`: main orchestration playbook
- `inventories/deploy/hosts.ini`: inventory groups for `dev` and `prod`
- `inventories/deploy/group_vars/all.yml`: shared variables such as deploy user, Node version, Git auth behavior, and Let's Encrypt defaults
- `inventories/deploy/group_vars/dev.yml`: environment-specific app definitions for dev
- `inventories/deploy/group_vars/prod.yml`: environment-specific app definitions for prod
- `roles/`: reusable automation units
- `vault/`: local secret and environment files referenced by inventory vars
- `AGENTS.md`: repository-specific operating notes for future coding sessions

## Inventory Model

Shared variables are defined in `inventories/deploy/group_vars/all.yml`.
This currently includes:

- `developer_user`
- `developer_home`
- `github_username`
- `github_persist_auth_in_remote`
- `nvm_version`
- `node_versions`
- `node_default_version`
- `letsencrypt_email`
- `letsencrypt_staging`

### Node version invariant

`node_versions` is the **install manifest** (the `nodejs` role installs each
version in the list). `node_default_version`, and any per-app `node_version`
override, are **selectors** used at runtime by `pm2`, `app_deploy`, and
`vite_app_deploy` (`nvm use` + PATH). Therefore:

> Every version referenced by `node_default_version` or a per-app
> `node_version` **must** also appear in `node_versions`, otherwise it is never
> installed and the deploy fails when it tries to `nvm use` a missing version.

This is enforced by a fail-fast `assert` at the start of the `nodejs` role
(tagged `prereq`, `deploy`, `app_code`, `app_pm2`, `vite_deploy`, `vite_code` —
deliberately **not** `letsencrypt`, so certificate-only runs skip the check).
A mismatch aborts early with a clear `MISSING: <version>` message instead of a
cryptic `nvm` error mid-deploy.

Environment-specific application definitions live in:

- `inventories/deploy/group_vars/dev.yml`
- `inventories/deploy/group_vars/prod.yml`

Two app collections are supported:

- `apps`: PM2-backed applications
- `vite_apps`: static Vite applications

Typical `apps` fields:

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

Typical `vite_apps` fields:

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

## Git Authentication Strategy

Repository checkout uses authenticated HTTPS URLs built from:

- `github_username`
- `github_token` loaded from `vault/github_token.yml`

The clone strategy is:

1. clone or update using an authenticated HTTPS remote URL
2. optionally reset `origin` back to the clean repository URL when `github_persist_auth_in_remote: false`

If `github_persist_auth_in_remote: true`, the authenticated remote remains in `.git/config` on the server.

## Nginx And TLS Behavior

`roles/nginx` installs nginx and shared include config, but final site vhosts are owned by deploy roles:

- `roles/app_deploy/templates/vhost.conf-http.j2`
- `roles/app_deploy/templates/vhost.conf-https.j2`
- `roles/vite_app_deploy/templates/vhost.conf-http.j2`
- `roles/vite_app_deploy/templates/vhost.conf-https.j2`

For PM2-backed apps:

- if `ssl_fullchain` and `ssl_privkey` are defined, the SSL vhost template is rendered
- otherwise, the HTTP-only vhost template is rendered

For Let's Encrypt:

- `roles/letsencrypt` installs `certbot` and `python3-certbot-nginx`
- it builds temporary challenge vhosts per domain
- it requests certificates with `certbot certonly --nginx`
- it removes the temporary challenge vhosts after issuance

## Quick Start

1. Update inventory hosts in `inventories/deploy/hosts.ini`.
2. Set shared variables in `inventories/deploy/group_vars/all.yml`.
3. Define your applications in `inventories/deploy/group_vars/dev.yml` or `inventories/deploy/group_vars/prod.yml`.
4. Create `vault/github_token.yml` and store `github_token` in it with Ansible Vault.
5. Create any environment files referenced by `env_src`, for example under `vault/dev/` or `vault/prod/`.
6. Run the playbook.

Examples:

```bash
ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --check
```

```bash
ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags prereq --limit "prod:dev"
```

```bash
ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags letsencrypt --limit prod
```

```bash
ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags deploy --limit dev
```

```bash
ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags vite_deploy --limit dev
```

Stage-scoped deploys (see the sub-phase tags above):

```bash
# Deploy code + PM2 but leave the nginx vhost untouched
ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags app_code,app_pm2 --limit dev
```

```bash
# Recreate only the PM2 process (no clone/build, no nginx)
ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags app_pm2 --limit dev
```

```bash
# Refresh only the nginx vhost for the deployed apps
ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --tags app_nginx --limit dev
```

`github_token` content example:
```
github_token: <the token value>
```
Then encrypt the value using the `ansible-vault` encrypt command:

```bash
ansible-vault encrypt vault/github_token.yml
```

For another encryption such as .env value under `vault/` directory:

example:

```bash
ansible-vault encrypt vault/prod/example-node-prod.env
```

```bash
ansible-vault encrypt vault/dev/example-node-dev.env
```


## Security Notes

- Never commit real secrets from `vault/`.
- Keep `vault/github_token.yml` local, ignored, and encrypted with Ansible Vault.
- Use `vault/github_token.example.yml` only as a public placeholder.
- Tasks that render authenticated clone URLs or `.env` files are intentionally marked with `no_log: true`.
- If `github_persist_auth_in_remote` is enabled, understand that token-bearing remote URLs remain on the target host in the Git config.

## Validation Notes

Useful checks:

- `ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --syntax-check`
- `ansible-playbook -i inventories/deploy/hosts.ini playbooks/site.yml --check`
- tag-scoped runs with `--limit dev` or `--limit prod`

If local Ansible execution depends on a vault password file, ensure `ansible.cfg` and your local environment are aligned before running checks.
