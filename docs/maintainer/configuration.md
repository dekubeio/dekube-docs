# Configuration reference

All persistent configuration lives in `helmfile2compose.yaml`. This file is created on first run and preserved across re-runs. User edits are never overwritten.

## Full example

```yaml
name: my-platform
volume_root: ./data
extensions:
  caddy:
    email: admin@example.com

distribution_version: v3.1.0
depends:
  - keycloak
  - cert-manager==0.3.0
  - trust-manager

volumes:
  data-postgresql:
    driver: local
  myapp-data:
    host_path: app
  other:
    host_path: ./custom

exclude:
  - prometheus-operator
  - meet-celery-*

replacements:
  - old: 'path_style_buckets = false'
    new: 'path_style_buckets = true'

overrides:
  redis-master:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "$secret:redis:redis-password"]
    volumes: ["$volume_root/redis:/data"]
    environment: null

services:
  minio-init:
    image: quay.io/minio/mc:latest
    restart: on-failure
    entrypoint: ["/bin/sh", "-c"]
    command:
      - mc alias set local http://minio:9000 $secret:minio:rootUser $secret:minio:rootPassword
        && mc mb --ignore-existing local/my-bucket
```

## Sections

### `name`

Compose project name. Auto-set to the source directory basename on first run.

### `volume_root`

Base path for volume host mounts. Default: `./data`.

Bare names in `host_path` are prefixed with this value. Paths starting with `./` or `/` are used as-is.

```yaml
volume_root: ./data

volumes:
  app-data:
    host_path: app        # resolves to ./data/app
  other:
    host_path: ./custom   # used as-is (starts with ./)
```

On first run, discovered PVCs are auto-registered with `host_path: <pvc_name>` (e.g. `host_path: data-postgresql` -> `./data/data-postgresql`). On subsequent runs, PVCs not mapped in the config emit a warning but are still usable as named Docker volumes.

### `volumes`

Map PVCs to named volumes or host paths.

```yaml
volumes:
  data-postgresql:
    driver: local          # named docker volume (no host path)
  myapp-data:
    host_path: app         # bind mount under volume_root
```

- `driver: local` — named Docker volume (managed by the runtime)
- `host_path: <name>` — bind mount. Bare name = `volume_root/<name>`. Explicit path (`./` or `/` prefix) = used as-is.

### `exclude`

Skip workloads by name. Supports wildcards via `fnmatch` syntax (`*`, `?`, `[seq]`).

```yaml
exclude:
  - prometheus-operator     # exact name
  - meet-celery-*           # wildcard
  - cert-manager-*          # pattern
```

On first run, K8s-only workloads matching `cert-manager`, `ingress`, `reflector` are auto-excluded.

Separately, workloads with `replicas: 0` are auto-skipped (with a warning), regardless of the `exclude` list. This check runs independently — a workload with `replicas: 0` is skipped even if it's not in `exclude`.

### `replacements`

Global find/replace applied to generated ConfigMap/Secret files, environment variable values, and Caddyfile upstreams. Port remapping is automatic — use replacements for other rewrites.

```yaml
replacements:
  - old: 'path_style_buckets = false'
    new: 'path_style_buckets = true'
  - old: 'some.old.hostname'
    new: 'new.hostname'
```

Replacements are also applied to Caddyfile upstream names.

### `overrides`

Deep merge into generated services. Useful for replacing images, commands, or environment variables.

```yaml
overrides:
  redis-master:
    image: redis:7-alpine
    command: ["redis-server", "--requirepass", "$secret:redis:redis-password"]
    volumes: ["$volume_root/redis:/data"]
    environment: null       # null deletes the key entirely
```

- **Deep merge**: nested dicts are merged recursively (e.g. you can override individual environment variables without replacing the entire `environment` block).
- **`null` deletes**: setting a key to `null` removes it from the generated service.
- Overrides that reference a non-existent generated service are skipped with a warning.

### `services`

Custom services not from K8s manifests. Added to compose output alongside generated services.

```yaml
services:
  minio-init:
    image: quay.io/minio/mc:latest
    restart: on-failure
    entrypoint: ["/bin/sh", "-c"]
    command:
      - mc alias set local http://minio:9000 $secret:minio:rootUser $secret:minio:rootPassword
        && mc mb --ignore-existing local/my-bucket
```

Combined with `restart: on-failure`, this is the brute-force retry pattern for one-shot init tasks (bucket creation, API configuration, etc.). The service retries until it succeeds, then exits.

If a custom service name conflicts with a generated service, the custom one overwrites it (with a warning).

### `extensions.caddy.email`

If set, generates a global Caddy block for automatic HTTPS certificate provisioning:

```
{
    email admin@example.com
}
```

Previously `caddy_email` (top-level). Migrated automatically on first run — see [Config migration](#config-migration).

### `extensions.caddy.tls_internal`

If `true`, adds `tls internal` to all Caddyfile host blocks. Forces Caddy to use its internal CA for all domains (useful for `.local` development domains).

Previously `caddy_tls_internal` (top-level). Migrated automatically on first run.

### `distribution_version`

Pin the distribution version for [h2c-manager](h2c-manager.md). Ignored by helmfile2compose itself.

```yaml
distribution_version: v3.1.0
```

`core_version` is accepted as a backwards-compatible alias.

### `distribution`

Select which distribution h2c-manager installs. Default: `helmfile2compose`. Use `core` for the bare engine (`h2c.py`).

```yaml
distribution: helmfile2compose
```

### `depends`

List of [h2c extensions](../catalogue.md) required by this project. h2c-manager reads this list and installs them automatically.

```yaml
depends:
  - keycloak
  - cert-manager==0.3.0
  - trust-manager
```

Bare names pull the latest release. Pin with `==version` for reproducibility (recommended — see [Your project](your-project.md#recommended-workflow)).

See [h2c-manager — declarative dependencies](h2c-manager.md#declarative-dependencies) for override behavior and details.

### `ingress_types`

Maps custom `ingressClassName` values to canonical rewriter names. Without this, custom class names won't match any rewriter and the Ingress is skipped with a warning.

```yaml
ingress_types:
  haproxy-controller-internal: haproxy
  haproxy-controller-external: haproxy
  nginx-internal: nginx
```

The mapping is applied before rewriter dispatch. See [Ingress controllers](your-project.md#ingress-controllers) for details.

Previously `ingressTypes` (camelCase). Migrated automatically on first run.

### `disable_ingress`

If `true`, skips the ingress provider service in `compose.yml`. Ingress rules are still written to `Caddyfile-<project>` for manual merging with your existing reverse proxy.

**Manual only** — never auto-generated. See [Advanced](../user/advanced.md) for the full cohabitation guide.

Previously `disableCaddy`. Migrated automatically on first run.

### `network`

Override the default compose network with an external one:

```yaml
network: shared-infra
```

Generates:

```yaml
networks:
  default:
    name: shared-infra
    external: true
```

Required when sharing a network across multiple compose projects. See [Advanced](../user/advanced.md).

### Config migration {#config-migration}

v3.1.0 normalized config keys to snake_case. On first run, old keys are automatically migrated:

| Old key | New key |
|---------|---------|
| `disableCaddy` | `disable_ingress` |
| `ingressTypes` | `ingress_types` |
| `caddy_email` | `extensions.caddy.email` |
| `caddy_tls_internal` | `extensions.caddy.tls_internal` |
| `helmfile2ComposeVersion` | removed (silently ignored) |

A stderr notice is printed if migration occurred. Migration happens in memory — to persist the new key names, edit your `helmfile2compose.yaml` manually (or delete it and re-run to regenerate from scratch).

## Placeholders

### `$secret:<name>:<key>`

Resolved from K8s Secret manifests at generation time. Usable in `overrides` and `services` values.

```yaml
overrides:
  my-service:
    environment:
      DB_PASSWORD: $secret:postgresql:password
```

The referenced Secret must exist in the rendered K8s manifests. Both `data` (base64-decoded) and `stringData` (plain) are supported.

### `$volume_root`

Resolved to the `volume_root` config value. Usable in `overrides` and `services` values.

```yaml
overrides:
  redis-master:
    volumes: ["$volume_root/redis:/data"]
# With volume_root: ./data, resolves to ./data/redis:/data
```
