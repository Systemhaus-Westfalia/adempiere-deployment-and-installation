# Known Issues & Technical Debt

## ⚠ Action Required Before Running

These two items will cause immediate failure or lock you out if not addressed first:

**1. Remove `custom_sshport` from the vault if it appears twice.**
All variables live in the single `group_vars/all.yml` vault file. If `custom_sshport` was added there more than once, Ansible will behave unpredictably. Edit the vault and ensure it appears only once:
```bash
ansible-vault edit group_vars/all.yml
# Ensure custom_sshport appears only once
```

**2. Verify the SSH port before re-running `serversconf.yml`.**
The port in `override.yml` is now `10099`. The servers were previously configured with port `42895` (from the old vault). If you run `serversconf.yml` with the new port, it will change the SSH port on the live servers. Make sure your firewall allows port `10099` before doing so, or you will be locked out.

---

## 1. Typo in `genkey.yml` — `connection: loca1`

**File:** `genkey.yml`, line 3
**Problem:** `connection: loca1` uses the digit `1` instead of the letter `l`. Should be `connection: local`.
**Current behavior:** Ansible warns about an unknown connection plugin and falls back to its default, which happens to work correctly for localhost. The play succeeds.
**Fix:** Change `loca1` → `local` in `genkey.yml`.

---


## 4. `serverswap` role is empty

**File:** `roles/serverswap/tasks/main.yml`
**Problem:** The role exists and has the directory structure but contains no tasks. It is not referenced by any playbook.
**Status:** Placeholder — swap space configuration is not yet implemented.

---

## 5. MOTD deployment is commented out

**File:** `roles/serversconf/tasks/main.yml`, lines 172–179
**Problem:** The task that deploys the `/etc/motd` banner from `motd.j2` is commented out. The template exists but is never deployed.
**Fix:** Uncomment the task block if the MOTD is desired.

---

## 6. Traefik dashboard has no authentication

**File:** `roles/deploy-traefik/templates/traefik.yaml.j2`
**Problem:** `api.insecure: true` exposes the dashboard on port `28080` without any authentication.
**Risk:** Anyone who can reach `<FrontEnd-IP>:28080` can see full routing configuration.
**Fix:** Either remove the dashboard in production (`traefik_dashboard_enabled: false`) or protect it with a middleware.

---

## 7. Plaintext Cloudflare credentials in `deploy-traefik/vars/main.yml`

See [security.md](security.md) — this is the most urgent item before pushing to GitHub.

---

## 8. `cloudflare_tocken` variable name is misspelled

**File:** `roles/deploy-traefik/vars/main.yml` and `roles/deploy-traefik/templates/.env.j2`
**Problem:** `cloudflare_tocken` (typo) instead of `cloudflare_token`.
**Fix:** Rename consistently in both files when moving the value to the vault.

---

---

## 9. Admin user password needs to be changed before production use

**File:** `roles/serversconf/vars/main.yml` — variable `your_password`  
**Problem:** The current SHA-512 hash is a placeholder/test password. It must be replaced with a
strong password before deploying to any production server.  
**Fix:** Generate a new hash and update the vault:
```bash
# Generate a new SHA-512 hash (you will be prompted for the password)
mkpasswd --method=sha-512

# Edit the vault file and replace your_password with the new hash
ansible-vault edit roles/serversconf/vars/main.yml
```

---

## 10. Future improvement — split `group_vars/all.yml` into vault + override

**Current state:** All variables (vault secrets and deployment-specific values such as IPs, domain, SSH port) are stored together in the single encrypted file `group_vars/all.yml`.

**Better practice:** Split into two files under `group_vars/all/`:
- `vault.yml` — AES-256 encrypted secrets only (passwords, API keys)
- `override.yml` — plain-text deployment values (IPs, domain, SSH port) — gitignored
- `override_template.yml` — committed template operators copy to create their own `override.yml`

This separation makes it easier to see which values need to change per deployment without decrypting the vault. Not urgent — the current single-file approach works correctly.

---

[← Troubleshooting](troubleshooting.md) | [Next: Security →](security.md)
