# Architecture

## Pipeline

The dark and twisted ritual underlying the conversion. The same Helm charts used for Kubernetes are rendered into standard K8s manifests, then converted to compose:

```
Helm charts (helmfile / helm / kustomize)
    |  helmfile template / helm template / kustomize build
    v
K8s manifests (Deployments, Services, ConfigMaps, Secrets, Ingress...)
    |  helmfile2compose.py
    v
compose.yml + Caddyfile + configmaps/ + secrets/
```

A dedicated helmfile environment (e.g. `compose`) typically disables K8s-only infrastructure (cert-manager, ingress controller, reflector) and adjusts defaults for compose.

## Converter dispatch

Manifests are classified by `kind` and dispatched to converter classes. Each converter handles one or more K8s kinds and returns a `ConvertResult` (compose services, Caddy entries, or both).

```
K8s manifests
    | parse + classify by kind
    | dispatch to converters (built-in or external, same interface)
    v
compose.yml + Caddyfile
```

### Built-in converters

- **`WorkloadConverter`** — kinds: DaemonSet, Deployment, Job, StatefulSet
- **`IngressConverter`** — kinds: Ingress

### External converters (operators)

Loaded via `--extensions-dir`. Each `.py` file (or one-level subdirectory with `.py` files) is scanned for classes with `kinds` and `convert()`. Loaded converters are sorted by `priority` (lower = earlier, default 100) and registered into the dispatch loop.

```
.h2c/extensions/
├── keycloak.py                        # flat file
├── h2c-operator-certmanager/          # cloned repo
│   ├── certmanager.py                 # converter class
│   └── requirements.txt
```

See [Writing operators](writing-operators.md) for the full guide.

## What it converts

| K8s kind | Compose equivalent |
|----------|-------------------|
| DaemonSet / Deployment / StatefulSet | `services:` (image, env, command, volumes, ports). Init containers become separate services with `restart: on-failure`. Sidecar containers become separate services with `network_mode: container:<main>` (shared network namespace). DaemonSet treated identically to Deployment (single-machine tool). |
| Job | `services:` with `restart: on-failure` (migrations, superuser creation). Init containers converted the same way. |
| ConfigMap / Secret | Resolved inline into `environment:` + generated as files for volume mounts |
| Service (ClusterIP) | Hostname rewriting (K8s Service name -> compose service name) |
| Service (ExternalName) | Resolved through alias chain (e.g. `docs-media` -> minio) |
| Service (NodePort / LoadBalancer) | `ports:` mapping |
| Ingress | Caddy service + Caddyfile `reverse_proxy` blocks. Path-rewrite annotations -> `uri strip_prefix`. Backend SSL -> Caddy TLS transport. |
| PVC / volumeClaimTemplates | Host-path bind mounts (auto-registered in `helmfile2compose.yaml`) |
| securityContext (runAsUser) | Auto-generated `fix-permissions` service (`chown -R <uid>`) for non-root bind mounts |

### Not converted (warning emitted)

- CronJobs
- Resource limits / requests, HPA, PDB

### Silently ignored (no compose equivalent)

- RBAC, ServiceAccounts, NetworkPolicies, CRDs (unless claimed by a loaded operator), IngressClass, Webhooks, Namespaces
- Probes (liveness, readiness, startup) — no healthcheck generation
- Unknown kinds trigger a warning

## Processing pipeline

1. **Parse manifests** — recursive `.yaml` scan, multi-doc YAML split, classify by kind. Malformed YAML files are skipped with a warning.
2. **Index lookup data** — ConfigMaps, Secrets, Services indexed for resolution during conversion.
3. **Build alias map** — K8s Service name -> workload name mapping. ExternalName services resolved through chain.
4. **Build port map** — K8s Service port -> container port resolution (named ports resolved via container spec).
5. **Pre-register PVCs** — from both regular volumes and `volumeClaimTemplates`.
6. **First-run init** — auto-exclude K8s-only workloads, generate default config.
7. **Dispatch to converters** — each converter receives its kind's manifests + a `ConvertContext`. External operators run in priority order (lower first), then built-in converters.
8. **Post-process env** — DNS rewriting, alias resolution, port remapping, replacements applied to all service environments (idempotent — catches operator-produced services).
9. **Apply overrides** — deep merge from config `overrides:` and `services:` sections.
10. **Hostname truncation** — services with names >63 chars get explicit `hostname:`.
11. **Write output** — `compose.yml`, `Caddyfile`, config/secret files.

## Automatic rewrites

These happen transparently during conversion:

- **K8s DNS** — `<svc>.<ns>.svc.cluster.local` and `<svc>.<ns>.svc` rewritten to compose service names. Applied to env vars, ConfigMap files, and Caddyfile upstreams.
- **Service aliases** — K8s Services whose name differs from the workload are resolved. ExternalName services followed through the chain.
- **Port remapping** — K8s Service port -> container port in URLs. `http://svc` (implicit port 80) and `http://svc:80` both rewritten to `http://svc:8080` if the container listens on 8080.
- **Kubelet `$(VAR)`** — `$(VAR_NAME)` in container command/args resolved from the container's env vars.
- **Shell `$VAR` escaping** — `$VAR` in command/entrypoint escaped to `$$VAR` for compose.
- **String replacements** — user-defined `replacements:` from config applied to env vars, ConfigMap files, and Caddyfile upstreams.

## Docker/Compose gotchas

These are Docker/Compose limitations, not conversion limitations. See [Limitations](../limitations.md) for what gets lost in translation.

- **Large port ranges** — K8s with `hostNetwork` handles thousands of ports natively. Docker creates one iptables/pf rule per port, so a range like 50000-60000 (e.g. WebRTC) will kill your network stack. Reduce the range in your compose environment values (e.g. 50000-50100).
- **hostNetwork** — K8s pods can bind directly to the host network. In Compose, every exposed port must be mapped explicitly.
- **S3 virtual-hosted style** — AWS SDKs default to virtual-hosted bucket URLs (`bucket-name.s3:9000`). Compose DNS can't resolve dotted hostnames. Configure your app to use path-style access and use a `replacement` if needed.

### Handled automatically

- **Service port remapping** — K8s Service port -> container port rewriting in env vars, configmap files, and Caddyfile upstreams.
- **K8s `$(VAR)` in commands** — Kubelet-style variable interpolation in container command/args resolved at generation time.
- **Shell `$VAR` escaping** — Shell variable references in command/entrypoint escaped (`$$`) so compose doesn't substitute them from host env.
- **K8s DNS rewriting** — `<svc>.<ns>.svc.cluster.local` and `<svc>.<ns>.svc` rewritten to compose service names.
- **Bind mount permissions** — Non-root containers with PVC bind mounts get an auto-generated `fix-permissions` service.
- **Hostname truncation** — Services >63 chars get explicit `hostname:`.
