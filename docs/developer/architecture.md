# Architecture

## Pipeline

The dark and twisted ritual underlying the conversion. The same Helm charts used for Kubernetes are rendered into standard K8s manifests, then converted to compose:

```
Helm charts (helmfile / helm / kustomize)
    |  helmfile template / helm template / kustomize build
    v
K8s manifests (Deployments, Services, ConfigMaps, Secrets, Ingress...)
    |  helmfile2compose.py (distribution) / h2c.py (bare core)
    v
compose.yml + Caddyfile + configmaps/ + secrets/
```

A dedicated helmfile environment (e.g. `compose`) typically disables K8s-only infrastructure (cert-manager, ingress controller, reflector) and adjusts defaults for compose.

For the internal package structure, module layout, and build system, see [h2c-core](h2c-core.md).

## Converter dispatch

Manifests are classified by `kind` and dispatched to converter classes. Each converter handles one or more K8s kinds and returns a `ConverterResult` (ingress entries only) or `ProviderResult` (compose services + ingress entries).

```
K8s manifests
    | parse + classify by kind
    | dispatch to converters (built-in or external, same interface)
    v
compose.yml + Caddyfile
```

### Built-in converters (distribution)

The bare h2c-core has **no** built-in converters — all registries are empty. The [helmfile2compose](https://github.com/helmfile2compose/helmfile2compose) distribution bundles 7 built-in extensions via `_auto_register()`:

- **`ConfigMapIndexer`** / **`SecretIndexer`** / **`PvcIndexer`** / **`ServiceIndexer`** — index resources into `ctx` ([h2c-indexer-*](https://github.com/helmfile2compose))
- **`WorkloadConverter`** — kinds: DaemonSet, Deployment, Job, StatefulSet ([h2c-converter-workload](https://github.com/helmfile2compose/h2c-converter-workload))
- **`HAProxyRewriter`** — built-in ingress rewriter, haproxy + default fallback ([h2c-rewriter-haproxy](https://github.com/helmfile2compose/h2c-rewriter-haproxy))
- **`CaddyProvider`** — IngressProvider, produces a Caddy service + Caddyfile ([h2c-provider-caddy](https://github.com/helmfile2compose/h2c-provider-caddy))

Each lives in its own repo, referenced in `distribution.json`. The distribution assembles them at build time via h2c-manager.

### External extensions (providers and converters)

Loaded via `--extensions-dir`. Each `.py` file (or one-level subdirectory with `.py` files) is scanned for classes with `kinds` and `convert()`. Providers (keycloak, servicemonitor) produce compose services; converters (cert-manager, trust-manager) produce synthetic resources. Both share the same code interface and are sorted by `priority` (lower = earlier, default 100) and registered into the dispatch loop.

```
.h2c/extensions/
├── keycloak.py                        # flat file
├── h2c-converter-cert-manager/         # cloned repo
│   ├── cert_manager.py                # converter class
│   └── requirements.txt
```

See [Writing converters](extensions/writing-converters.md) for the full guide.

### External transforms

Loaded from the same `--extensions-dir` as converters. The loader distinguishes them automatically: classes with `transform()` and no `kinds` are transforms. Sorted by `priority` (lower = earlier, default 100). Run after all converters, aliases, overrides, and hostname truncation — they see the final output.

See [Writing transforms](extensions/writing-transforms.md) for the full guide.

### Ingress rewriters

Ingress annotation handling is dispatched through `IngressRewriter` classes. Each rewriter targets a specific ingress controller (identified by `ingressClassName` or annotation prefix) and translates its annotations into ingress entry dicts consumed by the configured `IngressProvider`.

The built-in `HAProxyRewriter` handles `haproxy` and empty/absent ingress classes, plus any manifest with `haproxy.org/*` annotations. It also acts as the default fallback when no `ingressClassName` is set — if no external rewriter matches first, HAProxy claims the manifest.

External rewriters are loaded from `--extensions-dir` alongside converters and transforms. A rewriter with the same `name` as a built-in one replaces it. Dispatch order: external rewriters first, then built-in.

See [Writing rewriters](extensions/writing-rewriters.md) for the full guide.

## What it converts

| K8s kind | Compose equivalent |
|----------|-------------------|
| DaemonSet / Deployment / StatefulSet | `services:` (image, env, command, volumes, ports). Init containers become separate services with `restart: on-failure`. Sidecar containers become separate services with `network_mode: container:<main>` (shared network namespace). DaemonSet treated identically to Deployment (single-machine tool). |
| Job | `services:` with `restart: on-failure` (migrations, superuser creation). Init containers converted the same way. |
| ConfigMap / Secret | Resolved inline into `environment:` + generated as files for volume mounts |
| Service (ClusterIP) | Network aliases (FQDN variants resolve via compose DNS) |
| Service (ExternalName) | Resolved through alias chain (e.g. `docs-media` -> minio) |
| Service (NodePort / LoadBalancer) | `ports:` mapping |
| Ingress | Caddy service + Caddyfile `reverse_proxy` blocks, dispatched to ingress rewriters by `ingressClassName`. Path-rewrite annotations -> `uri strip_prefix`. Backend SSL -> Caddy TLS transport. Rewriters can inject `extra_directives` for rate-limit, auth, headers, etc. |
| PVC / volumeClaimTemplates | Host-path bind mounts (auto-registered in `helmfile2compose.yaml`) |
| securityContext (runAsUser) | Auto-generated `fix-permissions` service (`chown -R <uid>`) for non-root bind mounts |

### Not converted (warning emitted)

- CronJobs
- Resource limits / requests, HPA, PDB

### Silently ignored (no compose equivalent)

- RBAC, ServiceAccounts, NetworkPolicies, CRDs (unless claimed by a loaded extension), IngressClass, Webhooks, Namespaces
- Probes (liveness, readiness, startup) — no healthcheck generation
- Unknown kinds trigger a warning

## Processing pipeline

1. **Parse manifests** — recursive `.yaml` scan, multi-doc YAML split, classify by kind. Malformed YAML files are skipped with a warning.
2. **Index lookup data** — ConfigMaps, Secrets, Services indexed for resolution during conversion.
3. **Build alias map** — K8s Service name -> workload name mapping. ExternalName services resolved through chain.
4. **Build port map** — K8s Service port -> container port resolution (named ports resolved via container spec). When the Service is missing from manifests, named ports fall back to a well-known port table (`http` → 80, `https` → 443, `grpc` → 50051).
5. **Pre-register PVCs** — from both regular volumes and `volumeClaimTemplates`.
6. **First-run init** — auto-exclude K8s-only workloads, generate default config.
7. **Dispatch to converters** — each converter receives its kind's manifests + a `ConvertContext`. Extensions run in priority order (lower first), then built-in converters. Within `IngressProvider`, each Ingress manifest is dispatched to the first matching `IngressRewriter` (by `ingressClassName` or annotation prefix).
8. **Post-process env** — port remapping and replacements applied to all service environments (idempotent — catches extension-produced services).
9. **Build network aliases** — for each K8s Service, add FQDN aliases (`svc.ns.svc.cluster.local`, `svc.ns.svc`, `svc.ns`) + short alias to the compose service's `networks.default.aliases`. FQDNs resolve natively via compose DNS — no hostname rewriting needed.
10. **Apply overrides** — deep merge from config `overrides:` and `services:` sections.
11. **Hostname truncation** — services with names >63 chars get explicit `hostname:`.
12. **Run transforms** — external post-processing hooks (if loaded). Transforms mutate `compose_services` and `ingress_entries` in place.
13. **Write output** — `compose.yml`, `Caddyfile`, config/secret files.

## Automatic rewrites

These happen transparently during conversion:

- **Network aliases** — each compose service gets `networks.default.aliases` with all K8s FQDN variants (`svc.ns.svc.cluster.local`, `svc.ns.svc`, `svc.ns`). FQDNs in env vars, ConfigMaps, and Caddyfile upstreams resolve natively via compose DNS — no hostname rewriting needed. This preserves cert SANs for HTTPS.
- **Service aliases** — K8s Services whose name differs from the workload are resolved. ExternalName services followed through the chain. The short K8s Service name is added as a network alias on the compose service.
- **Port remapping** — K8s Service port -> container port in URLs. `http://svc` (implicit port 80) and `http://svc:80` both rewritten to `http://svc:8080` if the container listens on 8080. FQDN variants (`svc.ns.svc.cluster.local:80`) are also matched.
- **Kubelet `$(VAR)`** — `$(VAR_NAME)` in container command/args resolved from the container's env vars.
- **Shell `$VAR` escaping** — `$VAR` in command/entrypoint escaped to `$$VAR` for compose.
- **String replacements** — user-defined `replacements:` from config applied to env vars, ConfigMap files, and Caddyfile upstreams.

## Docker/Compose gotchas

These are Docker/Compose limitations, not conversion limitations. See [Limitations](../limitations.md) for what gets lost in translation.

- **Large port ranges** — K8s with `hostNetwork` handles thousands of ports natively. Docker creates one iptables/pf rule per port, so a range like 50000-60000 (e.g. WebRTC) will kill your network stack. Reduce the range in your compose environment values (e.g. 50000-50100).
- **hostNetwork** — K8s pods can bind directly to the host network. In Compose, every exposed port must be mapped explicitly.
- **S3 virtual-hosted style** — AWS SDKs default to virtual-hosted bucket URLs (`bucket-name.s3:9000`). Compose DNS can't resolve dotted hostnames. Configure your app to use path-style access and use a `replacement` if needed.

These are not bugs. These are the terms and conditions you accepted when you decided to commit perjury upon the Kubernetes temple.
