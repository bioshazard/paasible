# paasible

Deploy Traefik reverse proxy with Docker stacks.

## Requirements

- Ansible 2.14+
- community.docker collection
- Docker and Docker Compose v2 on target hosts

## Install

```bash
ansible-galaxy install git+https://github.com/you/paasible
```

Or from a local checkout:

```bash
ansible-galaxy role install --force -p ./roles ./roles/paasible
```

## Role Variables

### Infrastructure (group_vars/all/paasible.yml)

| Variable | Default | Description |
|----------|---------|-------------|
| `paasible_subdomain_root` | `example.com` | Root domain for services |
| `paasible_acme_email` | `admin@example.com` | Email for Let's Encrypt |
| `paasible_tls_resolver` | `cloudflare` | TLS cert resolver name |
| `paasible_stacks_root` | `/opt/paasible` | Base directory for stacks |
| `paasible_traefik_enabled` | `true` | Enable bundled Traefik |
| `paasible_traefik_dashboard_host` | `traefik.{{ paasible_subdomain_root }}` | Traefik dashboard host |
| `paasible_traefik_cf_token` | `""` | Cloudflare DNS API token |
| `paasible_traefik_cf_email` | `""` | Cloudflare account email |

### Workloads (host_vars/myhost/paasible_stacks.yml)

| Variable | Description |
|----------|-------------|
| `paasible_stacks` | Dict of user-defined stacks |

## Inventory Structure

```
inventory/
├── group_vars/all/
│   └── paasible.yml          # infrastructure config
├── host_vars/
│   └── myhost/
│       ├── paasible_stacks.yml  # workload stacks
│       └── secrets.yml       # sensitive vars (use ansible-vault)
└── hosts                     # inventory file
```

### Example: group_vars/all/paasible.yml

```yaml
paasible_subdomain_root: example.com
paasible_acme_email: admin@example.com
paasible_traefik_enabled: true
```

### Example: host_vars/myhost/secrets.yml

```yaml
# Use ansible-vault to encrypt
paasible_traefik_cf_token: "your-cloudflare-token"
paasible_traefik_cf_email: "admin@example.com"
```

### Example: host_vars/myhost/paasible_stacks.yml

```yaml
ansible_host: 192.168.1.40
ansible_user: ansible

paasible_stacks:
  whoami:
    env: {}
  minio:
    env:
      MINIO_ROOT_USER: "{{ minio_root_user }}"
      MINIO_ROOT_PASSWORD: "{{ minio_root_password }}"
```

### x-paasible schema

Add `x-paasible` to your service in docker-compose.yml:

```yaml
services:
  whoami:
    image: traefik/whoami
    x-paasible:
      port: 80
      host: whoami.example.com
    # or with multiple routes:
    # x-paasible:
    #   routes:
    #     - port: 80
    #       host: whoami.example.com
```

## Example Playbook

```yaml
- name: Deploy paasible
  hosts: paasible_hosts
  become: true
  roles:
    - role: paasible
      paasible_stacks_dir: "{{ playbook_dir }}/../stacks"
```

## Stacks Directory

Place your stack compose files in a `stacks/` directory relative to your playbook:

```
playbooks/
stacks/
├── whoami/
│   └── docker-compose.yml
└── minio/
    └── docker-compose.yml
```

The role reads compose files from `paasible_stacks_dir` (passed in playbook).

## Usage

```bash
# Deploy
ansible-playbook -i inventory playbooks/deploy.yml

# Deploy specific host
ansible-playbook -i inventory playbooks/deploy.yml -l myhost

# Force recreate stacks
ansible-playbook -i inventory playbooks/deploy.yml -e paasible_stack_recreate=always
```
