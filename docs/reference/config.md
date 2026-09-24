---
description: "dekube.yaml reference: every configuration key for volume mappings, excludes, overrides, and reverse-proxy behavior."
---

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
# StatefulSet volumeClaimTemplates are keyed <vct>-<sts> (K8s names
# them <vct>-<sts>-<ordinal>; compose runs one replica).
volumes:
  my-pvc:
    host_path: ./data/my-pvc        # bind mount (most common)
  shared-data:                       # named volume (no host_path)
    driver: local
  data-my-statefulset:               # volumeClaimTemplate "data" on StatefulSet "my-statefulset"
    host_path: ./data/data-my-statefulset

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

# If false, skip namespace inference entirely. Manifests without
# metadata.namespace will stay namespace-less — no FQDN aliases.
# Useful if your charts already set namespaces on all resources.
# Default: true
infer_namespaces: true

# Map custom ingressClassName values to canonical rewriter names.
# Without this, a custom class only matches through a rewriter's
# annotation heuristic (e.g. nginx.ingress.kubernetes.io/* annotations).
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
    # Skip the Caddy compose service specifically (checked separately from
    # disable_ingress; neither flag stops ingress entries from being collected)
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
| `volumes` | `dict` | `{}` | PVC claim name → `{host_path: "..."}` mapping. Auto-populated on first run. `volumeClaimTemplate` claims are keyed `<vct>-<sts>`. Named volumes (no `host_path`) are added to compose `volumes:` top-level. |
| `exclude` | `list[str]` | `[]` | Workload names to skip. Supports `fnmatch` wildcards. |
| `replacements` | `list[dict]` | `[]` | String replacements: `[{old: "...", new: "..."}]`. Applied once to env vars, ConfigMap files, text Secret files, and reverse proxy upstreams. An entry with an empty or null `old` is skipped; a null `new` means `""`. |
| `disable_ingress` | `bool` | `false` | Skip the reverse proxy compose service. Ingress manifests are still dispatched to rewriters and the config file (e.g. Caddyfile) is still written, renamed to `Caddyfile-<project>` so it isn't picked up by accident. |
| `infer_namespaces` | `bool` | `true` | Infer missing `metadata.namespace` from sibling manifests and helmfile metadata. Set to `false` if your charts already set namespaces on all resources. See [namespace inference](../understand/engine.md#namespace-inference). |
| `ingress_types` | `dict[str, str]` | *(none)* | Custom `ingressClassName` → canonical rewriter name mapping. |
| `network` | `str` | *(none)* | External compose network name. |
| `overrides` | `dict` | *(none)* | Per-service compose overrides (deep-merged). |
| `services` | `dict` | *(none)* | Custom compose services (added verbatim). |
| `extensions` | `dict` | `{}` | Per-extension config, keyed by extension `name`. |

!!! note "Legacy bare `volumes:` key for `volumeClaimTemplates`"
    An existing `dekube.yaml` written before the `<vct>-<sts>` naming still works: the engine falls back to the bare `<vct>` key (the data path doesn't move) and warns `PVC '<vct>-<sts>': using legacy mapping '<vct>' — rename it to '<vct>-<sts>' in dekube.yaml`. If several StatefulSets fall back to the same legacy key, they'd share a data directory — the engine warns `PVC collision: a, b share legacy mapping '<vct>' (same data directory) — give each its own entry and host_path in dekube.yaml` instead. Both warnings only fire on non-first runs.

## Per-extension config (`extensions.*`)

Each extension — converter, provider, indexer, transform, rewriter — receives its own config section via `ctx.extension_config`. The engine resolves it automatically: an extension with `name = "caddy"` receives the contents of `extensions.caddy` from the config file. An absent or empty block (`caddy:` with nothing under it) is `{}`.

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

The `enabled` key is checked by the engine before each `convert()` / `transform()` call and before rewriter dispatch. When `false`, the extension is loaded but never executed; a disabled rewriter's Ingresses fall through to the next matching rewriter.

!!! note "`extensions.nginx` is shared"
    The nginx ingress provider and the nginx rewriter are both named `nginx`, so they read the same block — and `enabled: false` there disables both.

Empty entries are treated as absent rather than crashing the run: `overrides: {svc: }`, `services: {svc: }`, an empty `volume_root` (→ `./data`), an empty `ingress_types` value.

## Special value placeholders

Two placeholder patterns are resolved in config values:

- **`$volume_root`** — replaced with the value of `volume_root`. In `volumes:` entries it must lead the `host_path` (`$volume_root` or `$volume_root/…`); in `overrides:` and `services:` it's replaced anywhere:

    ```yaml
    volume_root: ./data
    volumes:
      my-pvc:
        host_path: $volume_root/my-pvc   # → ./data/my-pvc
    ```

- **`$secret:<secret_name>:<key>`** — resolved to the value of a K8s Secret key, in `overrides:`, `services:` and `replacements:`. The key ends at the first character a Secret key can't contain (anything outside `[-._a-zA-Z0-9]`), so `postgres://app:$secret:db:password@db:5432/app` keeps its `@db`. In `replacements:` the refs are resolved once the Secrets are indexed (generated ones included), right before the first provider runs; the resolved value is never written back to `dekube.yaml`:

    ```yaml
    replacements:
      - old: "PLACEHOLDER_PASSWORD"
        new: "$secret:my-secret:password"
    ```

## `$` escaping in `overrides:`

Every `environment` value the engine or an extension generates has its `$` doubled (`$$`) so compose passes it through literally instead of interpolating it — this runs once, after all transforms, and **before** `overrides:` are applied. `overrides:` values themselves stay raw on purpose: you keep compose's own `${VAR}` interpolation available there. The one exception is a `$secret:<name>:<key>` reference inside an override — it's still resolved and escaped, so the secret value itself arrives literal.

!!! note "Upgrading"
    If you were pre-escaping `$$` by hand in chart values or `replacements:` to work around the old behavior, you'll now get `$$$$` — remove the manual escaping. A `${VAR}` you meant for compose interpolation but that arrives through chart values or `replacements:` is now taken literally — move it into `overrides:` instead, where it's left raw.

## Upgrading from engine ≤ v1.7.0 {#upgrading-from-engine-v170}

Changes you can see after regenerating with a newer engine than v1.7.0 (helmfile2compose v3.4.0, kubernetes2simple v1.2.0):

- **Mounts with `items`** get their own `configmaps/<name>_<hash>/` (or `secrets/…`) directory, so their paths in `compose.yml` change. Update anything pointing at the old `configmaps/<name>/`. Mounts without `items` keep `<name>/`.
- **`env` wins over `envFrom`**, as in Kubernetes (and the last `envFrom` source wins). A variable defined in both used to take the `envFrom` value.
- **`$` in command/args**: every `$` is now escaped for compose, so `${X}` and `$$` reach the container's shell as written, and `$$(VAR)` is a literal `$(VAR)`. A `${VAR}` meant for compose interpolation in a container command must move to `overrides:`. Literal env values now get `$(VAR)` expansion, and `$$` in them becomes `$`, as with kubelet.
- **An extension that fails to load stops the run** (exit 1). It used to print a warning and convert without it — e.g. cert-manager with `cryptography` missing.
- **PVC `subPath` is honoured**: the mount moves to `<host_path>/<subPath>`. If the volume root already holds data and the subdirectory doesn't exist, the whole volume stays mounted with a warning until you move the data.
- **Secret files hold decoded bytes**: binary keys (keystores) used to be written as base64 text. A binary value referenced as an env var is skipped with a warning instead of passing base64.
- **A LoadBalancer Service publishes on its `port`**, not its `nodePort` (simple-workload after v0.4.0): the host port moves if the chart set a `nodePort`. NodePort Services still publish `nodePort`.
- **fix-permissions emulates `fsGroup`** (after v0.1.7): data directories of pods with an `fsGroup` get that group, `g+rwX` and setgid on directories — host-side modes on existing data change — and their services get `group_add` plus a `depends_on` on `fix-permissions`.
- **Caddy `server-ca`** (after v0.2.2): a CA Secret present in the manifests is now written to `./secrets/<name>/`. One that isn't (ExternalSecret, hand-placed files) keeps its mount, with a warning: put `ca.crt` in `./secrets/<name>/` yourself.
- **HAProxy is the ingress fallback** (haproxy rewriter after v0.1.3, priority 1100). With nginx or traefik loaded (kubernetes2simple, or `--extensions-dir`), a classless Ingress carrying their annotations now goes to them instead of getting plain HAProxy routing. `enabled: false` now disables rewriters too.
- **servicemonitor** (after v0.3.5): without `namespaceSelector`, only Services in the ServiceMonitor's own namespace match, as with the operator. Cross-namespace setups need `namespaceSelector`. Targets use the K8s Service name, and every `job_name` becomes `serviceMonitor/<ns>/<name>/<i>` as with the operator — the `job` label your dashboards and alerts query changes with it.
- **fake-apiserver** (after v0.666.2) requires the service-account token and binds its exposed port to `127.0.0.1`. Reconvert, then hand out the new kubeconfig.
- **Distribution builders**: `build-distribution.py` fails when two sources define a top-level name differently ([details](../understand/build-system.md#top-level-collisions)).

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
