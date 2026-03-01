# ansible-common

Reusable Ansible framework for homelab infrastructure. Targets Debian/Ubuntu servers.

## Architecture

```
site.yml                    # Main playbook — imports all roles with tags
roles/
  common/                   # Base packages, user setup, SSH keys, fail2ban
  docker/                   # Docker Engine + Compose plugin
  compose/                  # Generic stack deployment (dirs, env, compose up)
  systemd_timer/            # Scheduled tasks via systemd timer/service units
inventory/
  example/                  # Example inventory (copy and customize)
```

This repo is designed to be consumed by a **private overlay repo** via Git submodule or subtree. Compose files and secrets live in the private repo; this repo provides the automation framework.

## Roles

### `common`

Base server configuration.

| Variable | Default | Description |
|---|---|---|
| `common_packages` | `[curl, wget, git, htop, ...]` | APT packages to install |
| `common_user` | `deploy` | User to create |
| `common_user_shell` | `/bin/bash` | Login shell |
| `common_ssh_public_keys` | `[]` | SSH public keys to authorize |
| `common_fail2ban_enabled` | `true` | Install and enable fail2ban |

### `docker`

Installs Docker CE (includes Compose plugin) from the official Docker repository.

| Variable | Default | Description |
|---|---|---|
| `docker_edition` | `ce` | Docker edition |
| `docker_users` | `[]` | Users to add to docker group |

### `compose`

Deploys Docker Compose stacks. Compose files are copied as-is (no Jinja2 templating) — all variables go in `.env` files.

| Variable | Default | Description |
|---|---|---|
| `compose_stacks` | `[]` | List of stack definitions |
| `compose_base_dir` | `/opt/stacks` | Base directory for stacks |

Stack definition:

```yaml
compose_stacks:
  - name: mystack
    src: path/to/compose.yml
    env_src: path/to/.env        # optional
    env_dest: /opt/stacks/mystack/.env
```

### `systemd_timer`

Creates systemd timer/service units for scheduled tasks.

| Variable | Default | Description |
|---|---|---|
| `systemd_timer_timers` | `[]` | List of timer definitions |

Timer definition:

```yaml
systemd_timer_timers:
  - name: my-task
    description: "Run my task"
    command: "/usr/local/bin/my-script.sh"
    schedule: "*-*-* 02:00:00"   # OnCalendar format
    user: root                   # optional, default: root
```

## Quick Start

1. Install dependencies:

   ```bash
   ansible-galaxy install -r requirements.yml
   ```

2. Copy the example inventory:

   ```bash
   cp -r inventory/example inventory/mylab
   ```

3. Edit `inventory/mylab/hosts.yml` with your hosts and customize variables in `group_vars/` and `host_vars/`.

4. Run the playbook:

   ```bash
   ansible-playbook site.yml -i inventory/mylab/hosts.yml
   ```

   Or run specific roles with tags:

   ```bash
   ansible-playbook site.yml -i inventory/mylab/hosts.yml --tags docker,compose
   ```

## Using with a Private Overlay Repo

```bash
# In your private repo
git submodule add https://github.com/ma8el/ansible-common.git common

# Create your inventory and compose files alongside it
mylab/
  common/              # this repo (submodule)
  inventory/
    production/
      hosts.yml
      group_vars/all.yml
      host_vars/...
  stacks/
    traefik/
      docker-compose.yml
      .env
```

Point `compose_stacks[].src` at your local compose files in the private repo.

## Linting

```bash
yamllint .
ansible-lint
ansible-playbook site.yml --syntax-check -i inventory/example/hosts.yml
```
