# quadlet-atuin

Quadlet setup for the [Atuin](https://atuin.sh) shell history sync server (`ghcr.io/atuinsh/atuin:latest`).

This project was created with the help of Claude Code and https://github.com/mkoester/quadlet-my-guidelines/blob/main/new_quadlet_with_ai_assistance.md.

## Files in this repo

| File | Description |
|---|---|
| `atuin.container` | Quadlet unit file |
| `atuin.env` | Default environment variables |
| `atuin.override.env.template` | Template for local overrides |
| `atuin-backup.service` | Systemd service: creates SQLite backup via `sqlite3 .backup` |
| `atuin-backup.timer` | Systemd timer: triggers the backup daily |

## Setup

Run these commands from the root of this repository.

```sh
# 1. Create service user (regular user, home in /var/lib)
sudo useradd -m -d /var/lib/atuin -s /usr/sbin/nologin atuin

# 2. Enable linger
sudo loginctl enable-linger atuin

# 3. Create quadlet and data directories
sudo -u atuin mkdir -p ~atuin/.config/containers/systemd
sudo -u atuin mkdir -p ~atuin/config

# 4. Symlink .container and .env from the repo
sudo -u atuin ln -s $(pwd)/atuin.container ~atuin/.config/containers/systemd/atuin.container
sudo -u atuin ln -s $(pwd)/atuin.env ~atuin/.config/containers/systemd/atuin.env

# 5. Create .override.env from template (edit as needed)
sudo -u atuin cp $(pwd)/atuin.override.env.template ~atuin/.config/containers/systemd/atuin.override.env

# 6. Reload and start
sudo -u atuin systemctl --user daemon-reload
sudo -u atuin systemctl --user start atuin

# 7. Verify
sudo -u atuin systemctl --user status atuin
```

## Configuration

`atuin.env` contains the defaults:

| Variable | Default | Description |
|---|---|---|
| `ATUIN_HOST` | `0.0.0.0` | Bind address inside the container |
| `ATUIN_PORT` | `8888` | Port the server listens on |
| `ATUIN_OPEN_REGISTRATION` | `true` | Allow new user registrations |
| `ATUIN_DB_URI` | `sqlite:///config/atuin.db` | Database URI (SQLite in `~/config`) |

To override any value locally, edit `~atuin/.config/containers/systemd/atuin.override.env`.
For example, once all users have registered, disable open registration:

```sh
sudo -u atuin nano ~atuin/.config/containers/systemd/atuin.override.env
# Add: ATUIN_OPEN_REGISTRATION=false
sudo -u atuin systemctl --user restart atuin
```

## Backup

The backup writes a consistent SQLite snapshot to `/var/backups/atuin/` via `sqlite3 .backup` (online, no service downtime). Requires `sqlite3` to be installed on the host (`sudo dnf install sqlite` / `sudo apt install sqlite3`). A remote machine pulls it over SSH using the shared `backupuser`. See the [general backup setup](https://github.com/mkoester/quadlet-my-guidelines#backup) in the guidelines for the one-time server-wide setup (group, backup user, SSH key).

```sh
# 1. Create backup staging directory (owned by atuin, readable by backup-readers group)
sudo mkdir -p /var/backups/atuin
sudo chown atuin:backup-readers /var/backups/atuin
sudo chmod 750 /var/backups/atuin

# 2. Symlink the backup service and timer from the repo
sudo -u atuin mkdir -p ~atuin/.config/systemd/user
sudo -u atuin ln -s $(pwd)/atuin-backup.service ~atuin/.config/systemd/user/atuin-backup.service
sudo -u atuin ln -s $(pwd)/atuin-backup.timer ~atuin/.config/systemd/user/atuin-backup.timer

# 3. Enable and start the timer
sudo -u atuin systemctl --user daemon-reload
sudo -u atuin systemctl --user enable --now atuin-backup.timer
```

### On the remote (backup) machine

```sh
rsync -az backupuser@atuin-host:/var/backups/atuin/ /path/to/local/backup/atuin/
```

## Notes

- Port `8888` is bound to `127.0.0.1` only — place a reverse proxy in front for external access.
- The SQLite database is stored at `~atuin/config/atuin.db` on the host.
- `AutoUpdate=registry` is enabled; activate the timer once to get automatic image updates:
  ```sh
  sudo -u atuin systemctl --user enable --now podman-auto-update.timer
  ```
