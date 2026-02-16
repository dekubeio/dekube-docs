# Configuration reference

All persistent configuration lives in `helmfile2compose.yaml`. This file is created on first run and preserved across re-runs. User edits are never overwritten.

## Full example

```yaml
helmfile2ComposeVersion: v1
name: my-platform
volume_root: ./data
caddy_email: admin@example.com

core_version: v2.0.0
depends:
  - keycloak
  - certmanager==0.1.0
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

Auto-discovered PVCs default to `host_path: <pvc_name>` (e.g. `host_path: data-postgresql` -> `./data/data-postgresql`).

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

Workloads with `replicas: 0` are also auto-skipped (with a warning).

On first run, K8s-only workloads matching `cert-manager`, `ingress`, `reflector` are auto-excluded.

### `replacements`

Global find/replace applied to generated ConfigMap/Secret files and environment variable values. Port remapping is automatic — use replacements for other rewrites.

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

### `caddy_email`

If set, generates a global Caddy block for automatic HTTPS certificate provisioning:

```
{
    email admin@example.com
}
```

### `caddy_tls_internal`

If `true`, adds `tls internal` to all Caddyfile host blocks. Forces Caddy to use its internal CA for all domains (useful for `.local` development domains).

### `core_version`

Pin the h2c-core version. When `h2c-manager` runs, it uses this instead of fetching the latest release.

```yaml
core_version: v2.0.0
```

CLI `--core-version` overrides this.

### `depends`

List of [h2c extensions](../operators.md) required by this project. When `h2c-manager` is run with no explicit extensions on the command line, it reads this list and installs them automatically.

```yaml
depends:
  - keycloak
  - certmanager==0.1.0
  - trust-manager
```

Version pinning is supported with `==`. Explicit CLI extensions override the yaml list.

This turns `helmfile2compose.yaml` into the single source of truth for the project's dependencies — no hardcoded lists in shell scripts.

### `disableCaddy`

If `true`, skips the Caddy service in `compose.yml`. Ingress rules are still written to `Caddyfile-<project>` for manual merging with your existing reverse proxy.

**Manual only** — never auto-generated. See [Advanced](advanced.md) for the full cohabitation guide.

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

Required when sharing a network across multiple compose projects. See [Advanced](advanced.md).

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
