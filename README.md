# paasible

PaaS + Ansible. Deploy Traefik with Cloudflare SSL and Docker Compose stacks to your own servers.

Traefik is bundled and managed by the role. You define stacks as plain Docker Compose files — annotate a service with `x-paasible` and it gets routed automatically.

## Requirements

- Ansible 2.14+
- `community.docker` collection (`ansible-galaxy collection install community.docker`)
- Docker and Docker Compose v2 on target hosts
- A Cloudflare account with DNS API access

## Install

```bash
ansible-galaxy install git+https://github.com/bioshazard/paasible
```

## Quick Start

**1. Configure your inventory:**

```
inventory/
├── group_vars/all/
│   └── paasible.yml
├── host_vars/
│   └── myhost/
│       ├── paasible_stacks.yml
│       └── secrets.yml          # ansible-vault encrypted
└── hosts
```

**`group_vars/all/paasible.yml`:**
```yaml
paasible_subdomain_root: example.com
paasible_acme_email: admin@example.com
paasible_traefik_cf_token: "{{ cf_dns_api_token }}"
paasible_traefik_cf_email: "{{ cf_api_email }}"
```

**`host_vars/myhost/secrets.yml`** (encrypt with `ansible-vault encrypt`):
```yaml
cf_dns_api_token: "your-cloudflare-token"
cf_api_email: "admin@example.com"
```

**`host_vars/myhost/paasible_stacks.yml`:**
```yaml
paasible_stacks:
  whoami:
    env: {}
```

**2. Write a stack:**

```yaml
# stacks/whoami/docker-compose.yml
services:
  whoami:
    image: traefik/whoami:latest
    x-paasible:
      port: 80
      host: whoami.example.com
```

**3. Write your playbook:**

```yaml
- name: Deploy paasible
  hosts: paasible_hosts
  become: true
  roles:
    - paasible
```

**4. Run it:**

```bash
ansible-playbook -i inventory playbooks/deploy.yml
```

Traefik comes up first, your stacks follow. Certificates are issued automatically via Cloudflare DNS challenge.

---

## Role Variables

### Required

| Variable | Description |
|----------|-------------|
| `paasible_subdomain_root` | Root domain, e.g. `example.com` |
| `paasible_acme_email` | Email for Let's Encrypt |
| `paasible_traefik_cf_token` | Cloudflare DNS API token |
| `paasible_traefik_cf_email` | Cloudflare account email |

### Optional

| Variable | Default | Description |
|----------|---------|-------------|
| `paasible_stacks_root` | `/opt/paasible` | Where stacks are deployed on the host |
| `paasible_tls_resolver` | `cloudflare` | Traefik cert resolver name |
| `paasible_traefik_enabled` | `true` | Deploy bundled Traefik |
| `paasible_traefik_dashboard_host` | `traefik.{{ paasible_subdomain_root }}` | Traefik dashboard hostname |

---

## Stack Compose Convention

Stack files live in `stacks/<name>/docker-compose.yml` relative to your playbook directory. They are plain Docker Compose files — they run standalone without paasible.

Add `x-paasible` to any service you want Traefik to route:

```yaml
services:
  myapp:
    image: myapp:latest
    x-paasible:
      port: 8080
      host: myapp.example.com   # optional, defaults to {service}.{subdomain_root}
```

### Multiple routes per service

```yaml
services:
  minio:
    image: quay.io/minio/minio:latest
    x-paasible:
      routes:
        - port: 9000
          host: minio.example.com
        - port: 9001
          host: minio-console.example.com
```

### x-paasible schema

| Key | Type | Description |
|-----|------|-------------|
| `port` | integer | Single route shorthand |
| `host` | string | FQDN, defaults to `{service}.{subdomain_root}` |
| `routes` | list | Multiple routes, each with `port` and optional `host` |

---

## Traefik Dynamic Config

Drop `.yml` files into `stacks/traefik/dynamic/` to add middlewares, extra routers, or any other Traefik file provider config. Traefik hot-reloads this directory — no restart needed after a deploy.

---

## Secrets

Use `ansible-vault` to encrypt `secrets.yml`:

```bash
ansible-vault encrypt inventory/host_vars/myhost/secrets.yml
ansible-vault edit inventory/host_vars/myhost/secrets.yml
```

Run playbooks with vault decryption:

```bash
ansible-playbook -i inventory playbooks/deploy.yml --ask-vault-pass
# or
ansible-playbook -i inventory playbooks/deploy.yml --vault-password-file .vault-pass
```

---

## Usage

```bash
# Deploy everything
ansible-playbook -i inventory playbooks/deploy.yml

# Deploy to a specific host
ansible-playbook -i inventory playbooks/deploy.yml -l myhost

# Force recreate all stacks
ansible-playbook -i inventory playbooks/deploy.yml -e paasible_stack_recreate=always

# Dry run
ansible-playbook -i inventory playbooks/deploy.yml --check
```

---

## License

MIT