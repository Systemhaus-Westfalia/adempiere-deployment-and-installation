# Files Explained

Detailed explanations of individual project files — what each one does, why it is structured that way, and what to watch out for.  
Each section covers one file: its name, location, and a full description.

---

## roles/genkey/tasks/main.yml

**Name:** `main.yml`  
**Location:** `roles/genkey/tasks/main.yml`

**Description:**

The entry point for the `genkey` role, which is invoked by `genkey.yml`. It runs on `localhost` (the control node), not on any remote server. It contains four tasks:

**Task 0 — Create the `ssh_keys/` directory**  
Ensures `<project_root>/ssh_keys/` exists with mode `0700` before generating anything.

**Task 1 — Generate the keypair**  
Uses `community.crypto.openssh_keypair` to create the keypair at:
```
ssh_keys/adempiere_installation_key        (private — gitignored)
ssh_keys/adempiere_installation_key.pub    (public  — tracked by git)
```
The key name comes from the `key_name` variable (`roles/genkey/defaults/main.yml`); key size from `key_size` (default: 4096 bits).  
`state: present` means the task is idempotent: if the keypair already exists it is left untouched — no overwrite.

**Task 2 — Copy the public key into the role**  
Copies `ssh_keys/adempiere_installation_key.pub` to `roles/serversconf/files/public_keys/present/admin/<hostname>.pub`, using the control node's hostname (`ansible_facts['nodename']`) as the filename.  
The `serversconf` role picks up all `.pub` files from that directory via a glob and deploys them to the remote servers' `authorized_keys`.

**Task 3 — Confirm (debug)**  
Prints the path and comment of the generated key. Purely informational — no side effect.

**Behaviour under `--check` (dry run):**  
All four tasks support check mode.  
Task 1 reports `changed` if no key exists yet, `ok` if it does — without writing anything.  
Tasks 0 and 2 also simulate without writing.  
Task 3 always runs. The dry run is accurate for this role.

**Why this matters:**  
`genkey.yml` must be the first playbook run in a fresh deployment.  
Without the keypair in place, `serversconf` cannot populate `authorized_keys` and subsequent playbooks that connect as the `westfalia` user will fail authentication.

**Why a dedicated key inside the project (not `~/.ssh/id_rsa`):**  
Using the OpenSSH default `~/.ssh/id_rsa` is simpler (no configuration needed) but risky on a developer's workstation that already has an `id_rsa` for GitHub or personal SSH — `state: present` would silently reuse it.  
A dedicated named key inside the project is isolated, portable, and self-contained: the public key travels with the repository and a new operator only needs to run `genkey.yml` once after cloning.  
The private key is referenced via `ansible_ssh_private_key_file` in `group_vars/all.yml` so all playbooks pick it up automatically without any extra flags.

**`id_rsa` vs. dedicated key — trade-offs (documented for context):**

| | `~/.ssh/id_rsa` | `ssh_keys/adempiere_installation_key` (current) |
|---|---|---|
| Configuration needed | None — picked up automatically | `ansible_ssh_private_key_file` in `group_vars/all.yml` |
| Risk of reusing wrong key | Yes — silently reuses existing `id_rsa` | No — always the right key |
| Passphrase risk | Existing `id_rsa` may have one, breaking unattended runs | Generated without passphrase specifically for automation |
| Key isolation | Shared across all purposes | Independent — rotate or revoke without affecting anything else |
| Self-contained for GitHub | No | Yes — public key committed alongside the playbooks |

---

## serversprep.yml

**Name:** `serversprep.yml`  
**Location:** project root

**Description:**

The playbook that prepares a freshly provisioned server for all subsequent Ansible connections. It targets the `contabo` group (both BackEnd and FrontEnd) and must run before any other playbook that connects via SSH.

It does two things before invoking the role:

**pre_task 1 — Set connection credentials**  
Sets `ansible_user: root` and `ansible_password` from the vault (`root_ansible_password`). This is how Ansible connects to a server that has not yet been hardened — root login with password on port 22.

**pre_task 2 — Add server fingerprint to known_hosts**  
Runs `ssh-keyscan` against the server IP and writes the result to `~/.ssh/known_hosts` on the control node (`delegate_to: localhost`). Without this, SSH would prompt "unknown host" and the playbook would hang or fail.

Then calls the `serversprep` role.

**Why `gather_facts: false`:**  
Facts are gathered via SSH. On a brand-new server, the fingerprint is not yet in `known_hosts`, so an SSH connection would fail before facts could be collected. Setting `gather_facts: false` lets the `pre_tasks` handle fingerprinting first.

---

## roles/serversprep/tasks/main.yml

**Name:** `main.yml`  
**Location:** `roles/serversprep/tasks/main.yml`

**Description:**

Two tasks that run on the remote server after the playbook's `pre_tasks` have established the connection:

**Task 1 — Add fingerprint (remote side)**  
A second `known_hosts` call, this time running on the remote server rather than the control node. In practice the `pre_tasks` version (which runs on `localhost`) is the one that matters for Ansible connectivity; this task is redundant and may be removed in a future cleanup.

**Task 2 — Install the public key on the server**  
Adds `ssh_keys/adempiere_installation_key.pub` to root's `authorized_keys` on the remote server. After this step, all subsequent playbooks can authenticate as root using the project keypair instead of the vault password — and once `serversconf.yml` runs and disables password auth entirely, this key becomes the only way in.

**Key path:**  
Uses `playbook_dir + '/ssh_keys/' + key_name + '.pub'` — consistent with the `genkey` role. If `genkey.yml` has not been run first, this lookup will fail.

**Behaviour under `--check` (dry run):**  
`known_hosts` and `authorized_key` both support check mode and will report `changed` or `ok` without making changes. The dry run is accurate for this role.
