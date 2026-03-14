# paasible — Desired State

## Stack Identity

Each entry in `paasible_stacks` has an optional `template` key that resolves the compose file path (`stacks/{template}/docker-compose.yml`). When `template` is absent, the dict key is used as the template name (existing behavior, fully backward-compatible). The dict key is always the **stack identity** — it becomes the Docker Compose project name, the intra-stack network name, and the DNS prefix for cross-stack addressing.

```yaml
paasible_stacks:
  minio-a:
    template: minio
    external_networks:
      - shared-ops-net
    env:
      MINIO_ROOT_USER: "{{ minio_a_user }}"
  minio-b:
    template: minio
    external_networks:
      - shared-ops-net
    env:
      MINIO_ROOT_USER: "{{ minio_b_user }}"
```

---

## Network Architecture

Paasible manages three tiers of networks:

**1. Per-stack internal network (`{stack_key}-net`)**
Every stack gets a generated bridge network named `{stack_key}-net`. Created and owned by paasible, referenced as `external: true` by both Traefik and the stack overlay.

**2. Traefik network (`traefik`)**
A single shared bridge network owned by the `traefik` stack. Services do NOT join it — instead, Traefik discovers services via the Docker socket and uses the `traefik.docker.network={stack_key}-net` label to route traffic through the stack's internal network.

**3. Shared external networks**
Named networks declared in `paasible_stacks` under `external_networks`. Paasible collects the union of all declared external networks across all stacks, creates them before user stacks deploy, and removes them when the last consumer stack is removed.

---

## Overlay Generation

For each stack, paasible generates a `labels.yml` overlay (applied via Docker Compose file merging) that does the following:

**For every `x-paasible` service:**
- Joins `{stack_key}-net` (intra-stack)
- Adds `traefik.enable=true` and `traefik.docker.network={stack_key}-net` so Traefik reaches the service via the stack's internal network
- Adds all route labels (`routers`, `services`, `tls`) using `{stack_key}-{service}-{port}` as the router/service name

**For every service on a shared external network:**
- Joins the external network with the alias `{stack_key}-{service_name}`
- Cross-stack callers address this service as `{stack_key}-{service_name}` on the shared network — deterministic, no hashes

**Top-level `networks:` block:**
- Declares `{stack_key}-net` as external
- Declares each external network as external

---

## Stack Markers

Every deployed stack writes a JSON marker file at `{paasible_stacks_root}/{stack_key}/.paasible-stack`:

```json
{
  "stack": "minio-a",
  "external_networks": ["shared-ops-net"]
}
```

Markers serve two purposes:

**Removal detection:** At the start of each run, paasible finds all markers on disk. Any marker whose stack key is absent from the current `paasible_stacks` dict identifies a removed stack. The corresponding stack is stopped and its directory removed.

**External network lifecycle:** Markers are slurped before any directory removal so their `external_networks` data is available for cleanup. After stopping and removing a stack, paasible computes the orphaned external networks — those declared in removed stacks' markers that are no longer referenced by any remaining stack — and removes them. A shared network used by multiple stacks is never removed until the last consumer is gone.

Markers are written **after** a successful `docker_compose_v2` deploy, not before. A failed deploy produces no marker, which causes the next run to attempt cleanup of the partial state rather than treating it as managed.

Legacy markers (bare stack key string, written by older versions) are handled gracefully — they parse as having no external networks, meaning one transition run will not clean up external networks for stacks being removed in that run. Any stack that completes a normal deploy first will have a well-formed marker.

---

## Task Execution Order

```
Find managed stack markers
Compute remaining marker paths          ← pre-capture for diff
Slurp markers for removed stacks        ← capture ext_nets before deletion
Stop removed stacks
Remove removed stack directories
Collect all networks to manage
Create managed networks
Copy user stack compose files
Generate stack overlays
Write .env files
Deploy user stacks
Write stack markers                     ← after successful deploy
Deploy Traefik                          ← skipped when paasible_stacks is empty
Remove orphaned stack-nets
Remove orphaned external networks       ← diff removed vs still-referenced
Teardown path (when paasible_stacks: {})
```

**Traefik is not deployed during full teardown.** The Deploy Traefik block is guarded by both `paasible_traefik_enabled` and `paasible_stacks | length > 0`. When all stacks are removed, Traefik is stopped as part of the teardown path rather than being needlessly restarted.

---

## Lifecycle Invariants

- A user compose file never references `traefik` or any shared external network directly
- Every network has exactly one owner: paasible owns all bridge networks (stack + external), `traefik` owns its bridge
- No network is ever floating — all networks are created before stacks deploy and removed after the last dependent stack is gone
- A shared external network is never removed while any remaining stack declares it in `external_networks`
- The dict key uniquely identifies a deployment; deploying the same template twice requires two distinct keys, which guarantees no DNS alias collisions on shared networks
- Traefik is never started or restarted during a full teardown run

---

## x-paasible Schema

`external_networks` is declared at the **stack level** in `paasible_stacks`, not in the compose file. The compose file remains the portable service definition; network topology is an infrastructure concern owned by paasible.

```yaml
services:
  minio:
    image: quay.io/minio/minio
    x-paasible:
      routes:
        - port: 9000
          host: minio.example.com
        - port: 9001
          host: minio-console.example.com
```

Stack-level declaration in inventory:

```yaml
paasible_stacks:
  minio-a:
    template: minio
    external_networks:
      - shared-ops-net
    env:
      MINIO_ROOT_USER: "{{ minio_a_user }}"
```