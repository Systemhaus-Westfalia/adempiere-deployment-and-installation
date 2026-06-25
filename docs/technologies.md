# Technologies Used

## Table of Contents

- [Ansible](#ansible)
  - [Concrete example — playbook calling a role](#concrete-example--playbook-calling-a-role)
- [Traefik](#traefik)
- [Docker & Docker Compose](#docker--docker-compose)

---

## Ansible

Ansible is an agentless IT automation tool.  
You write **playbooks** (YAML files) that describe the desired state of your infrastructure.  
Ansible  
- connects to target machines over SSH  
- executes the tasks, and  
- ensures the system matches the declared state — without installing anything on the remote servers.

Key concepts used in this project:

| Concept | What it is |
|---|---|
| **Playbook** | A YAML file defining a sequence of tasks to run on a set of hosts |
| **Role** | A reusable, self-contained unit of tasks, templates, and variables |
| **Inventory** | A file listing the servers Ansible manages, organized into groups |
| **Vault** | Ansible's built-in encryption for storing secrets (passwords, tokens) |
| **Template** | A Jinja2 file rendered on the fly with variable substitution before being deployed |
| **Handler** | A task that runs only when triggered by a `notify`, typically to restart a service |

**Official documentation:** https://docs.ansible.com/

### Concrete example — playbook calling a role

A **playbook** is the entry point. It says *where* to connect and *what role* to run:

```yaml
# install-docker.yml
- name: Install Docker
  hosts: servers          # target group from inventories/hosts.yml
  become: true            # use sudo on the remote server
  roles:
    - install-docker      # delegate all work to this role
```

Running it:
```bash
ansible-playbook install-docker.yml
```

Ansible reads the inventory, opens SSH connections to all hosts in the `servers` group, and hands control to the `install-docker` role.

---

A **role** is a self-contained directory that does the actual work. The layout is fixed — Ansible knows where to look for each type of file:

```
roles/install-docker/
├── tasks/
│   └── main.yml        ← the steps to execute, in order
├── defaults/
│   └── main.yml        ← variable defaults (lowest priority — always overridable)
└── vars/
    └── main.yml        ← role constants (higher priority — rarely overridden)
```

The tasks file is where the work happens:

```yaml
# roles/install-docker/tasks/main.yml
- name: Install Docker packages
  ansible.builtin.apt:
    name:
      - docker-ce
      - docker-ce-cli
      - docker-compose-plugin
    state: present

- name: Enable Docker service
  ansible.builtin.service:
    name: docker
    state: started
    enabled: true
```

Each task uses an Ansible **module** (`apt`, `service`, `copy`, `template`, …) that knows how to make a specific change and — crucially — how to check whether the change is already in place. If Docker is already installed, the `apt` task reports `ok` and does nothing. This is **idempotency**: you can run the same playbook ten times and it only changes what is not yet in the desired state.

---

**How the pieces connect in this project:**

```
inventories/hosts.yml      defines which servers exist and their IPs
        │
        ▼
install-docker.yml         selects the "servers" group, calls the role
        │
        ▼
roles/install-docker/      does the actual work on each server over SSH
```

The playbook contains no task logic — it is only the connector between *where* (inventory group) and *what* (role). All logic lives inside the role.

---

**Official documentation:** https://docs.ansible.com/

---

## Traefik

Traefik is a modern reverse proxy and load balancer designed for containerized environments. Unlike traditional proxies (nginx, HAProxy), Traefik discovers services automatically by watching the Docker daemon, and can obtain and renew TLS certificates from Let's Encrypt without any manual configuration.

Key concepts used in this project:

| Concept | What it is |
|---|---|
| **EntryPoint** | A network port Traefik listens on (`:80`, `:443`) |
| **Router** | A rule that matches incoming requests (e.g. by hostname) and forwards them to a service |
| **Service** | The backend that handles matched requests (an IP address and port) |
| **Middleware** | Optional processing applied to requests/responses (headers, redirects, auth) |
| **CertificateResolver** | Configuration for automatic TLS certificate issuance via ACME (Let's Encrypt) |
| **DNS Challenge** | A way to prove domain ownership by creating a DNS TXT record — used here via the Cloudflare API, so no public HTTP port is required for cert issuance |
| **Socket Proxy** | A separate container that gives Traefik read-only access to the Docker API, preventing a compromised Traefik from controlling the Docker daemon |

**Official documentation:** https://doc.traefik.io/traefik/

---

## Docker & Docker Compose

Docker packages applications into containers. Docker Compose defines multi-container stacks in a single `docker-compose.yml` file.

This project installs Docker CE from the official Docker repository (not the distribution default) and uses the Compose plugin (`docker compose`, not the legacy `docker-compose`).

**Official documentation:** https://docs.docker.com/

---

[← Back to README](../README.md) | [Next: Requirements →](requirements.md)
