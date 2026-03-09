# Configuration reference (`dekube.yaml`)

> *The covenant was not written in stone but in clay — soft, yielding, shaped by the hand that first pressed it. Once fired, it held its form, and no disciple could reshape what the kiln had sealed.*
>
> — *Necronomicon, On Covenants That Harden (debatable)*

`dekube.yaml` is the project config file, created automatically on first run in `--output-dir`. After creation, the engine reads it but never writes to it — it's your file. Edit it freely.

The engine also accepts the legacy name `helmfile2compose.yaml` (with a deprecation warning).

## Full schema

```yaml
# Project name — used as compose file's top-level `name:` field.
# Auto-detected from helmfile on first run.
name: my-project

# Root directory for PVC bind mounts. PVC host_path values
# containing $volume_root are resolved relative to this.
# Default: ./data
volume_root: ./data

# PVC volume mappings. Auto-populated on first run.
# Each key is a PVC claim name from the K8s manifests.
volumes:
  my-pvc:
    host_path: ./data/my-pvc        # bind mount (most common)
  shared-data:                       # named volume (no host_path)
    driver: local

# Workload names to exclude from conversion.
# Auto-populated on first run with K8s-only workloads
# (cert-manager, reflector, ingress controllers...).
# Supports fnmatch wildcards (e.g. "cert-manager-*").
exclude:
  - cert-manager
  - cert-manager-cainjector
  - cert-manager-webhook
  - haproxy-*

# User-defined string replacements applied to env vars,
# ConfigMap files, Secret files, and reverse proxy upstreams.
replacements:
  - old: "https://app.prod.example.com"
    new: "https://app.localhost"

# If true, skip the reverse proxy service in compose.yml but still
# write the ingress config as Caddyfile-<project> (for external use).
# Use when you manage your own reverse proxy outside of dekube.
# Default: false
# Legacy key: disableCaddy
disable_ingress: false

# Map custom ingressClassName values to canonical rewriter names.
# Without this, custom class names won't match any rewriter.
# Legacy key: ingressTypes
ingress_types:
  haproxy-controller-internal: haproxy
  haproxy-controller-external: haproxy
  nginx-internal: nginx

# External compose network name. When set, the generated
# compose.yml uses an external network instead of creating one.
network: my-existing-network

# Per-service compose overrides. Deep-merged into the generated
# service definition. Use to patch image, env, volumes, etc.
overrides:
  my-service:
    image: custom-image:latest
    environment:
      EXTRA_VAR: "value"
    volumes:
      - ./custom:/app/custom:ro

# Custom compose services added verbatim to the output.
# Use to inject services that don't come from K8s manifests.
services:
  maildev:
    image: maildev/maildev:latest
    restart: always
    ports:
      - "1080:1080"

# Per-extension configuration. Each key matches an extension's
# `name` attribute. Extensions read this via ctx.extension_config.
extensions:
  caddy:
    # Disable the Caddy service entirely (different from disable_ingress:
    # this only affects Caddy, not the ingress entry collection)
    disabled: false
    # ACME email for Let's Encrypt
    email: admin@example.com
    # Use Caddy's internal CA instead of Let's Encrypt
    tls_internal: true
  # Disable an extension without removing it from --extensions-dir
  my-extension:
    enabled: false
  # Extension-specific keys (varies per extension)
  bitnami:
    # ...bitnami-specific config
```

## Key reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `name` | `str` | *(auto-detected)* | Compose project name. Set from helmfile on first run. |
| `volume_root` | `str` | `./data` | Root directory for PVC bind mount paths. |
| `volumes` | `dict` | `{}` | PVC claim name → `{host_path: "..."}` mapping. Auto-populated on first run. Named volumes (no `host_path`) are added to compose `volumes:` top-level. |
| `exclude` | `list[str]` | `[]` | Workload names to skip. Supports `fnmatch` wildcards. |
| `replacements` | `list[dict]` | `[]` | String replacements: `[{old: "...", new: "..."}]`. Applied to env vars, ConfigMap files, and reverse proxy upstreams. |
| `disable_ingress` | `bool` | `false` | Skip reverse proxy generation entirely. |
| `ingress_types` | `dict[str, str]` | *(none)* | Custom `ingressClassName` → canonical rewriter name mapping. |
| `network` | `str` | *(none)* | External compose network name. |
| `overrides` | `dict` | *(none)* | Per-service compose overrides (deep-merged). |
| `services` | `dict` | *(none)* | Custom compose services (added verbatim). |
| `extensions` | `dict` | `{}` | Per-extension config, keyed by extension `name`. |

## Per-extension config (`extensions.*`)

Each extension receives its own config section via `ctx.extension_config`. The engine resolves it automatically: an extension with `name = "caddy"` receives the contents of `extensions.caddy` from the config file.

```yaml
extensions:
  caddy:
    email: admin@example.com    # → ctx.extension_config["email"]
    tls_internal: true          # → ctx.extension_config["tls_internal"]
```

Any extension can be disabled without removing it from `--extensions-dir`:

```yaml
extensions:
  my-extension:
    enabled: false
```

The `enabled` key is checked by the engine before each `convert()` / `transform()` call. When `false`, the extension is loaded but never executed.

## Special value placeholders

Two placeholder patterns are resolved in config values:

- **`$volume_root`** — replaced with the value of `volume_root`. Useful in `volumes:` entries:

    ```yaml
    volume_root: ./data
    volumes:
      my-pvc:
        host_path: $volume_root/my-pvc   # → ./data/my-pvc
    ```

- **`$secret:<secret_name>:<key>`** — resolved to the value of a K8s Secret key. Useful in `replacements:` when you need to inject a Secret value into a string:

    ```yaml
    replacements:
      - old: "PLACEHOLDER_PASSWORD"
        new: "$secret:my-secret:password"
    ```

## Legacy key migration

On load, the engine auto-migrates legacy keys to their modern equivalents:

| Legacy key | Migrated to |
|------------|-------------|
| `disableCaddy` | `disable_ingress` |
| `ingressTypes` | `ingress_types` |
| `caddy_email` | `extensions.caddy.email` |
| `caddy_tls_internal` | `extensions.caddy.tls_internal` |
| `helmfile2ComposeVersion` | *(removed)* |

Migration happens in memory on load. The file is not rewritten — rename the keys manually when convenient.

## Related

- [CLI reference](cli.md) — command-line flags
- [Writing extensions](../extend/extensions/index.md) — how extensions read `ctx.extension_config`
- [Pitfalls](../pitfalls.md) — null-safe YAML access patterns
