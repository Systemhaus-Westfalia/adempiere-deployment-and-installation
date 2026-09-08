# serversconf

Full hardening and base configuration of a freshly provisioned Linux server.

Runs as part of `serversconf.yml` — after `serversprep.yml` (SSH key distribution) and `os-updates.yml` (OS update + reboot).

---

## What it does

1. Updates the apt cache and installs ~35 utility packages.
2. Ensures the configured locale is present.
3. Sets the system timezone.
4. Sets the server hostname.
5. Creates the admin user with a hashed password and bash shell.
6. Adds the admin user to `sudo`; grants passwordless sudo.
7. Deploys `.bashrc` to root and the admin user from `templates/bashrc.j2`.
8. Deploys SSH public keys from `files/public_keys/present/admin/` to both users.
9. Configures unattended-upgrades.
10. Moves SSH from port 22 to the custom port via a systemd socket override.
11. Deploys `sshd_config.d/01-hardening.conf` — disables root login and password auth, sets modern crypto.
12. Deploys `/etc/motd` from `templates/motd.j2` using the `motd_header` variable.

---

## Variables

### Mandatory (set in `group_vars/all/vars.yml` or `vault.yml`)

| Variable | Description |
|---|---|
| `adempiere_username` | Admin user to create. All post-hardening playbooks connect as this user. |
| `custom_sshport` | SSH port after hardening (replaces 22). |
| `server_hostname` | Hostname to assign, replacing the provider default. |
| `server_locale` | System locale (e.g. `en_US.UTF-8`). |
| `timezone` | System timezone (e.g. `America/El_Salvador`). |
| `root_user_password` | *(vault)* Root password for initial connection. |
| `adempiere_user_password` | *(vault)* SSH login password for the admin user. |
| `adempiere_user_become_pass` | *(vault)* sudo password for the admin user. |
| `your_password` | *(serversconf/vars/main.yml)* SHA-512 hashed password for the admin user. |

### Optional (role default in `defaults/main.yml`, override in `group_vars/all/vars.yml`)

| Variable | Default | Description |
|---|---|---|
| `motd_header` | ACME Inc (figlet small) | Multi-line ASCII art written to `/etc/motd`. See below. |

---

## MOTD customisation (`motd_header`)

The default is a generic "ACME Inc" placeholder. Override it in `group_vars/all/vars.yml` with your own art.

**How to generate the art:**

```bash
figlet -f small 'My Company'
```

**How to set it in `vars.yml`:**

If the first line of the art has a different number of leading spaces than the other lines (common with figlet small), use an explicit YAML indent indicator `|2` and add exactly 2 spaces to every line:

```yaml
motd_header: |2
    __  __         ___
   |  \/  |_  _   / __|___ _ __  _ __  __ _ _ _ _  _
   | |\/| | || | | (__/ _ \ '  \| '_ \/ _` | ' \ || |
   |_|  |_|\_, |  \___\___/_|_|_| .__/\__,_|_||_\_, |
           |__/                 |_|             |__/
```

YAML strips exactly 2 spaces from every line, restoring the original art.

You can verify the result before running Ansible:

```bash
python3 -c "
import yaml
with open('group_vars/all/vars.yml') as f:
    data = yaml.safe_load(f)
print(data['motd_header'])
"
```

---

## License

MIT-0
