# Architecture

## Pipeline

The dark and twisted ritual underlying the conversion. The same Helm charts used for Kubernetes are rendered into standard K8s manifests, then converted to compose:

```
Helm charts (helmfile / helm / kustomize)
    |  helmfile template / helm template / kustomize build
    v
K8s manifests (Deployments, Services, ConfigMaps, Secrets, Ingress...)
    |  helmfile2compose.py (distribution) / dekube.py (bare core)
    v
compose.yml + Caddyfile + configmaps/ + secrets/
```

A dedicated helmfile environment (e.g. `compose`) typically disables K8s-only infrastructure (cert-manager, ingress controller, reflector) and adjusts defaults for compose.

For the internal package structure, module layout, and build system, see [dekube-engine](dekube-engine.md).

## Converter dispatch

Manifests are classified by `kind` and dispatched to converter classes. Each converter handles one or more K8s kinds and returns a `ConverterResult` (ingress entries only) or `ProviderResult` (compose services + ingress entries).

**`kinds` is immutable.** The `Converter` base class declares `kinds: tuple = ()`. Extensions override it as a class attribute — this replaces the default entirely, which is the intended pattern. The base default is an empty tuple (not a list) to prevent accidental mutation: if an extension forgot to override `kinds` and called `self.kinds.append(...)`, a mutable default would silently corrupt the shared class attribute across all converter instances. A tuple makes this fail loudly.

```
K8s manifests
    | parse + classify by kind
    | dispatch to converters (built-in or external, same interface)
    v
compose.yml + Caddyfile
```

### The Eight Monks (distribution)

The bare dekube-engine has **no** built-in converters — all registries are empty. The [helmfile2compose](https://github.com/dekubeio/helmfile2compose) distribution bundles 8 bundled extensions via `_auto_register()`:

- **`ConfigMapIndexer`** / **`SecretIndexer`** / **`PvcIndexer`** / **`ServiceIndexer`** — index resources into `ctx` ([dekube-indexer-*](https://github.com/dekubeio))
- **`SimpleWorkloadProvider`** — kinds: DaemonSet, Deployment, Job, StatefulSet ([dekube-provider-simple-workload](https://github.com/dekubeio/dekube-provider-simple-workload))
- **`HAProxyRewriter`** — built-in ingress rewriter, haproxy + default fallback ([dekube-rewriter-haproxy](https://github.com/dekubeio/dekube-rewriter-haproxy))
- **`CaddyProvider`** — IngressProvider, produces a Caddy service + Caddyfile ([dekube-provider-caddy](https://github.com/dekubeio/dekube-provider-caddy))
- **`FixPermissions`** — transform, generates fix-permissions service for non-root bind mounts ([dekube-transform-fix-permissions](https://github.com/dekubeio/dekube-transform-fix-permissions))

Each lives in its own repo, referenced in `distribution.json`. The distribution assembles them at build time via dekube-manager.

### External extensions (providers and converters)

Loaded via `--extensions-dir`. Each `.py` file (or one-level subdirectory with `.py` files) is scanned for classes with `kinds` and `convert()`. Providers (keycloak, servicemonitor) produce compose services; converters (cert-manager, trust-manager) produce synthetic resources. Both share the same code interface and are sorted by `priority` (lower = earlier, default 100) and registered into the dispatch loop.

```
.dekube/extensions/
├── keycloak.py                        # flat file
├── dekube-converter-cert-manager/      # cloned repo
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
| PVC / volumeClaimTemplates | Host-path bind mounts (auto-registered in `helmfile2compose.yaml` on first run only) |
| securityContext (runAsUser) | Auto-generated `fix-permissions` service (`chown -R <uid>`) for non-root bind mounts (via the [fix-permissions](https://github.com/dekubeio/dekube-transform-fix-permissions) transform) |

### Not converted (warning emitted)

- CronJobs
- Resource limits / requests, HPA, PDB

### Silently ignored (no compose equivalent)

- RBAC, ServiceAccounts, NetworkPolicies, CRDs (unless claimed by a loaded extension), IngressClass, Webhooks, Namespaces
- Probes (liveness, readiness, startup) — no healthcheck generation
- Unknown kinds trigger a warning

## Processing pipeline

Thirteen steps. Each one locally reasonable. Together, they flatten a distributed system into a YAML file and a prayer.

1. **Parse manifests** — recursive `.yaml` scan, multi-doc YAML split, classify by kind. Malformed YAML files are skipped with a warning.
2. **Index lookup data** — ConfigMaps, Secrets, Services indexed for resolution during conversion.
3. **Build alias map** — K8s Service name -> workload name mapping. ExternalName services resolved through chain.
4. **Build port map** — K8s Service port -> container port resolution (named ports resolved via container spec). When the Service is missing from manifests, named ports fall back to a well-known port table (`http` → 80, `https` → 443, `grpc` → 50051).
5. **Track PVCs** — from both regular volumes and `volumeClaimTemplates`. On first run, auto-register in config for host_path mapping. On subsequent runs, track only (config is read-only after creation).
6. **First-run init** — auto-exclude K8s-only workloads, generate default config, write `helmfile2compose.yaml`. On subsequent runs: detect stale volume entries (config volumes not referenced by any PVC).
7. **Dispatch to converters** — each converter receives its kind's manifests + a `ConvertContext`. Extensions run in priority order (lower first), then built-in converters. Within `IngressProvider`, each Ingress manifest is dispatched to the first matching `IngressRewriter` (by `ingressClassName` or annotation prefix).
8. **Post-process env** — port remapping and replacements applied to all service environments (idempotent — catches extension-produced services).
9. **Build network aliases** — for each K8s Service, add FQDN aliases (`svc.ns.svc.cluster.local`, `svc.ns.svc`, `svc.ns`) + short alias to the compose service's `networks.default.aliases`. FQDNs resolve natively via compose DNS — no hostname rewriting needed.
10. **Apply overrides** — deep merge from config `overrides:` and `services:` sections.
11. **Hostname truncation** — services with names >63 chars get explicit `hostname:`.
12. **Run transforms** — post-processing hooks (if loaded). Transforms mutate `compose_services` and `ingress_entries` in place.
13. **Write output** — `compose.yml`, `Caddyfile`, config/secret files. The temple is rendered. The architect goes to sleep. The architect does not sleep well.

## Automatic rewrites

These happen transparently during conversion:

- **Network aliases** — each compose service gets `networks.default.aliases` with all K8s FQDN variants (`svc.ns.svc.cluster.local`, `svc.ns.svc`, `svc.ns`). FQDNs in env vars, ConfigMaps, and Caddyfile upstreams resolve natively via compose DNS — no hostname rewriting needed. This preserves cert SANs for HTTPS.
- **Service aliases** — K8s Services whose name differs from the workload are resolved. ExternalName services followed through the chain. The short K8s Service name is added as a network alias on the compose service.
- **Port remapping** — K8s Service port -> container port in URLs. `http://svc` (implicit port 80) and `http://svc:80` both rewritten to `http://svc:8080` if the container listens on 8080. FQDN variants (`svc.ns.svc.cluster.local:80`) are also matched.
- **Kubelet `$(VAR)`** — `$(VAR_NAME)` in container command/args resolved from the container's env vars.
- **Shell `$VAR` escaping** — `$VAR` in command/entrypoint escaped to `$$VAR` for compose.
- **String replacements** — user-defined `replacements:` from config applied to env vars, ConfigMap files, and Caddyfile upstreams.

## Beyond single-host : Docker Swarm {#beyond-single-host}

The helmfile2compose distribution targets a single Docker host — bind mounts, no replicas, Caddy as a standalone reverse proxy, fix-permissions assuming a local filesystem. This is a distribution choice, not an engine limitation.

dekube-engine produces a service dict and dumps it as YAML — it doesn't validate or restrict what keys extensions put in. A provider can write `deploy.replicas`, `deploy.placement`, or any other section, and it will pass through to the output unchanged. The contracts (`Converter`, `Provider`, `IngressRewriter`, `ConvertContext`) have nothing mono-host-specific.

The format situation is unclear. Swarm and Compose both use YAML files with the same `deploy:` key for replicas, placement, and rolling updates — but they don't use the same spec. The [Swarm docs](https://docs.docker.com/reference/cli/docker/stack/deploy/) say `docker stack deploy` uses the legacy Compose v3 format and that the current Compose Specification "isn't compatible". The [Compose Deploy Specification](https://docs.docker.com/reference/compose-file/deploy/) documents `deploy:` as part of the current spec, supported by `docker compose`. Two pages, same domain, easy to misread. [Docker's own docs AI](https://docs.docker.com/) confirmed: Swarm is stuck on legacy v3. Docker backported Swarm's `deploy:` key into the Compose Specification without upgrading Swarm to support the Compose Specification — so the key exists in both formats, but the surrounding spec diverged. Swarm still wants `version: "3.x"` in the header; the Compose Spec dropped the `version:` field entirely. A Swarm distribution would need to produce Compose v3. A Swarm-oriented distribution would need different monks:

| Concern | Current monk (single-host) | Swarm equivalent |
|---------|---------------------------|------------------|
| Volumes | Bind mounts (`host_path`) | Distributed volume drivers (NFS, GlusterFS) or named volumes with `driver_opts` |
| Replicas | Ignored (single instance) | `deploy.replicas`, placement constraints, rolling updates |
| Reverse proxy | Caddy (standalone) | Traefik in Swarm mode (service mesh, automatic service discovery) |
| Permissions | fix-permissions (local `chown`) | Likely unnecessary (volume drivers handle ownership) or different strategy |
| Ingress | Single Caddyfile | Traefik labels on services, Let's Encrypt via Traefik |

The core engine, indexers, and most transforms would carry over unchanged. The providers and the ingress stack are where the opinions live.

Whether this is a good idea is a separate question. Swarm runs on a spec its own maintainers deprecated without ever upgrading — a format whose future is unclear at best, powering a proclaimed production-grade orchestrator. If you need multi-node scheduling, service mesh, and rolling updates, you already have Kubernetes. The case for helmfile2compose is clear: run the same apps on a single machine without the cluster overhead. The case for writing dekube extensions for Swarm is murkier — you'd be converting from one orchestrator to a less capable one. But the engine will not judge (it just reads YAML manifests, it produces a `.yml`). I — having built this abomination, pot, kettle, yadda — will.

## Docker/Compose gotchas

These are Docker/Compose limitations, not conversion limitations. See [Limitations](../limitations.md) for what gets lost in translation.

- **Large port ranges** — K8s with `hostNetwork` handles thousands of ports natively. Docker creates one iptables/pf rule per port, so a range like 50000-60000 (e.g. WebRTC) will kill your network stack. Reduce the range in your compose environment values (e.g. 50000-50100).
- **hostNetwork** — K8s pods can bind directly to the host network. In Compose, every exposed port must be mapped explicitly.
- **S3 virtual-hosted style** — AWS SDKs default to virtual-hosted bucket URLs (`bucket-name.s3:9000`). Compose DNS can't resolve dotted hostnames. Configure your app to use path-style access and use a `replacement` if needed.

These are not bugs. These are the terms and conditions you accepted when you decided to commit perjury upon the Kubernetes temple.
