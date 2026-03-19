# Extension catalogue

Extensions are external modules that extend helmfile2compose beyond its built-in capabilities. Install them via [dekube-manager](https://manager.dekube.io/docs/) or manually with `--extensions-dir`.

CRDs are K8s entities that don't speak compose — exiles from a world with controllers and reconciliation loops. The extension system is the immigration office: each converter forges the documents that the K8s controller would have produced at runtime. The forgery is disturbingly convincing.

## Extension types

There are six extension types, all loaded from the same `--extensions-dir`:

| Type | Interface | Purpose | Naming convention |
|------|-----------|---------|-------------------|
| **IndexerConverter** | `kinds` + `convert()` | Populate `ConvertContext` lookups (configmaps, secrets, PVCs, services) without producing services | `dekube-indexer-*` |
| **Converter** | `kinds` + `convert()` | Handle K8s resource kinds, produce synthetic resources (Secrets, ConfigMaps) | `dekube-converter-*` |
| **Provider** | `kinds` + `convert()` | Handle K8s resource kinds, produce compose services (and possibly resources) | `dekube-provider-*` |
| **Transform** | `transform()`, no `kinds` | Post-process the final compose output after converters | `dekube-transform-*` |
| **IngressProvider** | subclass of `IngressProvider`, `build_service()` + `write_config()` | Produce the reverse proxy service and config file from ingress entries | distribution-level |
| **Ingress rewriter** | `name` + `match()` + `rewrite()` | Translate ingress controller annotations into ingress entries | `dekube-rewriter-*` |

See [Writing extensions](extend/extensions/index.md) to build your own.

## The Eight Monks — bundled in helmfile2compose

These are the eight original extensions bundled in the [helmfile2compose distribution](understand/distributions.md). Together with the [emptydir transform](#emptydir), they form the reference distribution: everything you need to convert standard K8s manifests into a working compose stack, nothing you don't. No CRD magic, no vendor-specific annotations, no opinions about your monitoring stack — just Deployments, StatefulSets, Jobs, Services, Ingresses, ConfigMaps, Secrets, PVCs, and the plumbing to wire them together.

If your helmfile only uses standard Kubernetes resources, the monks and bundled transforms are all you need. The moment you introduce CRDs (cert-manager Certificates, Keycloak CRs, ServiceMonitors…) or vendor-specific ingress annotations (nginx, traefik), you install extensions from the sections below.

| Monk | Type | Kinds | Priority |
|------|------|-------|----------|
| [The Librarian](https://github.com/dekubeio/dekube-indexer-configmap) | IndexerConverter | `ConfigMap` | 50 |
| [The Guardian](https://github.com/dekubeio/dekube-indexer-secret) | IndexerConverter | `Secret` | 50 |
| [The Binder](https://github.com/dekubeio/dekube-indexer-pvc) | IndexerConverter | `PersistentVolumeClaim` | 50 |
| [The Weaver](https://github.com/dekubeio/dekube-indexer-service) | IndexerConverter | `Service` | 50 |
| [The Builder](https://github.com/dekubeio/dekube-provider-simple-workload) | Provider | `Deployment`, `StatefulSet`, `DaemonSet`, `Job` | 500 |
| [The Herald](https://github.com/dekubeio/dekube-rewriter-haproxy) | IngressRewriter | — | — |
| [The Gatekeeper](https://github.com/dekubeio/dekube-provider-caddy) | IngressProvider | `Ingress` | 900 |
| [The Custodian](https://github.com/dekubeio/dekube-transform-fix-permissions) | Transform | — | 8000 |

The four indexers populate `ConvertContext` lookups so that later stages can resolve ConfigMap keys, Secret references, PVC claims, and Service ports. The Builder turns workloads into compose services. The Herald translates HAProxy ingress annotations, the Gatekeeper assembles them into a Caddy reverse proxy. The Custodian runs last — it scans for non-root containers and generates a busybox init service that fixes bind mount permissions.

The distribution also bundles the [emptydir transform](#emptydir) (priority 1000), which auto-detects shared `emptyDir` volumes and promotes them to named Compose volumes. Not a monk — it was never part of the original engine extraction — but bundled from day one.

## Ingress providers

Ingress providers consume ingress entries (produced by rewriters) and generate the actual reverse proxy service + configuration file. The built-in Caddy provider ships with the distribution — it handles all ingress entries by default. If Caddy doesn't fit your setup, an Nginx provider is available as an alternative.

For building your own, see [Writing ingress providers](extend/extensions/writing-ingressproviders.md). The contract is straightforward: receive a list of ingress entries, return a compose service dict and write your config file.

### nginx-provider

| | |
|---|---|
| **Repo** | [dekube-provider-nginx](https://github.com/dekubeio/dekube-provider-nginx) |
| **Dependencies** | none |
| **Produces** | compose services + `nginx.conf` |
| **Status** | untested |

The alternative gate. Same ingress entries, different keeper. Generates an `nginx.conf` with upstream blocks, `proxy_pass` directives, and optional TLS (ACME via certbot, self-signed, or user-provided certs). Reads `response_headers`, `max_body_size`, `strip_prefix` from the structured entry format — no Caddy syntax leaks through.

Three TLS modes: `extensions.nginx.email` triggers certbot with ACME challenges, `extensions.nginx.tls_internal: true` generates self-signed certs via openssl, `extensions.nginx.tls_cert_path` mounts user-provided certificates. Without any TLS config, plain HTTP on port 80.

Untested in production. The Gatekeeper (Caddy) remains the recommended default. Use this if you need nginx specifically or if Caddy's automatic HTTPS doesn't suit your environment.

```bash
python3 dekube-manager.py nginx-provider
```

---

## Providers

Providers produce compose services — they emulate what a K8s controller would have created as running workloads. Install them via [dekube-manager](https://manager.dekube.io/docs/).

### keycloak

| | |
|---|---|
| **Repo** | [dekube-provider-keycloak](https://github.com/dekubeio/dekube-provider-keycloak) |
| **Kinds** | `Keycloak`, `KeycloakRealmImport` |
| **Dependencies** | none |
| **Priority** | 500 |
| **Produces** | compose services + realm JSON files |
| **Status** | stable |

Almost boring by this project's standards. The Keycloak Operator's job is to read a CR and produce a Deployment with the right env vars — which is exactly what dekube does for every other workload anyway. The `Keycloak` CR becomes a compose service with KC_* environment variables (database, HTTP, hostname, proxy, features). `KeycloakRealmImport` CRs are written as JSON files and mounted for auto-import on startup. A realm import is a static declaration that gets applied once — no reconciliation loop needed, no mutation, no drift to watch for. Turns out, removing the operator from a CRD that was already declarative just... works. Barely heretical.

Features: TLS secret mounting (if certs exist — doesn't care who made them), podTemplate volume support, bootstrap admin generation, realm placeholder resolution, namespace and K8s Service alias registration for network aliases.

```bash
python3 dekube-manager.py keycloak
```

---

### servicemonitor

| | |
|---|---|
| **Repo** | [dekube-provider-servicemonitor](https://github.com/dekubeio/dekube-provider-servicemonitor) |
| **Kinds** | `Prometheus`, `ServiceMonitor` |
| **Dependencies** | none |
| **Priority** | 600 |
| **Produces** | compose services + prometheus.yml |
| **Status** | stable |

Why would you need monitoring in a compose stack? You wouldn't. You absolutely wouldn't. And yet someone asked, and now here we are — a full Prometheus with auto-generated scrape targets, in docker compose, because apparently the shed needs an alarm system.

In Kubernetes, the Prometheus Operator watches ServiceMonitor CRDs and rewrites scrape config dynamically. This extension reads the same CRDs and bakes everything into a static `prometheus.yml`. No operator, no watch, no dynamic anything. Prometheus doesn't know the difference. Prometheus doesn't need to know.

Features: FQDN scrape targets (via network aliases), HTTPS scrape with CA bundle mounting (uses trust-manager ConfigMaps if available), named port resolution, label-based Service matching, fallback name-based matching for converter-created resources (e.g. Keycloak). No hard dependencies on other extensions — works standalone for HTTP scrape targets.

```bash
python3 dekube-manager.py servicemonitor
```

Grafana saw what we did to its lifelong companion and [fought back itself](https://helmfile2compose.dekube.io/docs/known-workarounds/kube-prometheus-stack/).

## Converters

Converters produce synthetic resources (Secrets, ConfigMaps, files on disk) without adding compose services. They forge the documents that a K8s controller would have produced at runtime — the resources exist, but no workload runs.

### cert-manager

| | |
|---|---|
| **Repo** | [dekube-converter-cert-manager](https://github.com/dekubeio/dekube-converter-cert-manager) |
| **Kinds** | `Certificate`, `ClusterIssuer`, `Issuer` |
| **Dependencies** | `cryptography` (Python package) |
| **Priority** | 100 |
| **Produces** | synthetic Secrets (PEM certificates) |
| **Status** | stable |

Very heretical — and paradoxically, the one with a strong case for existing. Try setting up a local CA chain, issuing certs with the right SANs for a dozen services, and mounting them where they need to go, all by hand in a compose file. cert-manager's declarative model actually makes *more* sense going through helmfile2compose than doing it manually. That's the uncomfortable part: the ICBM-to-kill-flies pipeline is, for once, genuinely simpler than the alternative.

This extension generates real PEM certificates at conversion time — CA chains, SANs, ECDSA/RSA — and injects them as synthetic K8s Secrets into the conversion context. No ACME challenges. No renewal. No controller. Just cryptographic material, conjured from nothing, valid until it isn't.

Then it starts merging certificates. Duplicate `secretName` entries across namespaces? Combined into a single cert with merged SANs. Rounds of issuance — self-signed CAs first, then CA-issued certs — because dependency order matters even in forgery. The kind of extension you can't predict, can't control, and can't entirely disapprove of — because the results speak for themselves, even if the methods are grounds for intervention.

```bash
python3 dekube-manager.py cert-manager
pip install cryptography  # required dependency
```

---

### trust-manager

| | |
|---|---|
| **Repo** | [dekube-converter-trust-manager](https://github.com/dekubeio/dekube-converter-trust-manager) |
| **Kinds** | `Bundle` |
| **Dependencies** | `cert-manager` extension; optional `certifi` (falls back to system CA paths) |
| **Priority** | 200 |
| **Produces** | synthetic ConfigMaps (CA bundles) |
| **Status** | stable |

The accomplice. Assembles CA trust bundles from cert-manager Secrets, ConfigMaps, inline PEM, and system default CAs. Injects the result as a synthetic ConfigMap. Pods that mount the trust bundle ConfigMap get the assembled CA chain automatically — believing they live in a cluster where a trust-manager controller reconciled this for them.

Depends on the cert-manager extension (needs its generated secrets). When installed via dekube-manager, cert-manager is auto-resolved as a dependency.

```bash
python3 dekube-manager.py trust-manager
# cert-manager is installed automatically as a dependency
```

## Transforms

Transforms are extensions that modify the final compose output *after* converters have run. They don't produce services from K8s manifests — they reshape what's already there. Whether this constitutes repair or further damage depends on your perspective.

### flatten-internal-urls

| | |
|---|---|
| **Repo** | [dekube-transform-flatten-internal-urls](https://github.com/dekubeio/dekube-transform-flatten-internal-urls) |
| **Dependencies** | none |
| **Priority** | 2000 |
| **Incompatible with** | `cert-manager` |
| **Status** | stable |

The only extension with a heresy score of NaN/10. It destroys the K8s naming temple — but in doing so, it reduces the overall desecration. Whether this makes it a sin or a penance is left as an exercise for the theologian.

Strips Docker Compose network aliases and rewrites K8s FQDNs (`svc.ns.svc.cluster.local`) to short compose service names. Rewrites env vars, configmap files on disk, and Caddy upstreams. v2.1 spent considerable effort building the alias system so that K8s names would carry into compose; this transform rips it all out. On purpose.

Built to restore **nerdctl compose** compatibility — nerdctl silently ignores network aliases, so FQDNs never resolve. But nerdctl isn't the only reason to use it. The transform also produces **cleaner compose output** — no 4-line alias blocks on every service, no `keycloak.auth.svc.cluster.local` in environment variables when `keycloak` would do. If you don't need FQDN preservation (no inter-service TLS, no cert SANs to match), flattening makes the generated files easier to read, debug, and diff.

**Incompatible with cert-manager**: the cert-manager extension generates certificates with SANs matching K8s FQDNs. Flattening those FQDNs breaks TLS verification. dekube-manager blocks this combination by default — use `--ignore-compatibility-errors` if you know what you're doing.

```bash
python3 dekube-manager.py flatten-internal-urls
```

---

### emptydir

| | |
|---|---|
| **Repo** | [dekube-transform-emptydir](https://github.com/dekubeio/dekube-transform-emptydir) |
| **Dependencies** | none |
| **Priority** | 1000 |
| **Status** | stable |

The quiet one. Kubernetes `emptyDir` volumes are shared scratch space between all containers in a pod — init containers prepare data, the main container consumes it, sidecars read it. In compose, pods become separate services, and separate services cannot share anonymous volumes (`- /path`). Each container would get its own independent anonymous volume, silently failing to share anything.

This transform auto-detects the problem: it scans all services for anonymous volume entries that originated from the same pod's `emptyDir`, identifies which ones are mounted by more than one service, and promotes them to named Compose volumes (`{workload}-{volume-name}`) declared at the top level. Services that shared the emptyDir in Kubernetes now share a named Compose volume — no configuration, no `dekube.yaml` entries, no manual intervention.

Uses `ctx.compose_extras` to declare the top-level `volumes:` block. Runs at priority 1000, before bitnami (1500) and fix-permissions (8000).

---

### bitnami

| | |
|---|---|
| **Repo** | [dekube-transform-bitnami](https://github.com/dekubeio/dekube-transform-bitnami) |
| **Dependencies** | none |
| **Priority** | 1500 |
| **Status** | stable |

The janitor. Bitnami charts — Redis, PostgreSQL, Keycloak — wrap standard images in custom entrypoints, init containers, and volume conventions that assume a full Kubernetes environment. In compose, the entrypoints fail, the volumes don't line up, and the init containers crash on missing emptyDirs. The workarounds are well-documented in [common charts](https://helmfile2compose.dekube.io/docs/known-workarounds/common-charts/) — this transform applies them automatically so you don't have to copy-paste overrides across projects.

Detects Bitnami images by name, then: replaces Redis entirely with stock `redis:7-alpine`, fixes PostgreSQL volume paths, injects Keycloak passwords as env vars and removes the failing init container. Every modification is logged to stderr. User `overrides:` take precedence — if you've already handled a service manually, the transform leaves it alone.

```bash
python3 dekube-manager.py bitnami
```

## Ingress rewriters

!!! note "Rewriter vs provider — two sides of the same gate"
    An **ingress rewriter** reads controller-specific annotations on each Ingress manifest and produces *ingress entries* (abstract routing rules: hostname, path, upstream, TLS). An **ingress provider** consumes those entries and builds the actual reverse proxy service + config file (e.g. Caddy). The rewriter answers *"what does `nginx.ingress.kubernetes.io/rewrite-target` mean?"* — the provider answers *"how do I generate a Caddyfile from these rules?"* They're orthogonal: you can swap the rewriter (nginx → traefik) without touching the provider, or swap the provider (Caddy → something else) without touching the rewriters.

Ingress rewriters translate controller-specific annotations into ingress entries consumed by the ingress provider. Unlike converters, they don't claim K8s kinds — they intercept individual Ingress manifests based on `ingressClassName` or annotation prefix, read whatever vendor-specific incantations the chart author scattered across the annotations, and produce routing rules that the provider assembles. The annotations were never meant to be portable. That's the point.

The built-in `HAProxyRewriter` handles `haproxy` and empty/absent ingress classes. If your cluster uses something else — and statistically, it does, because nobody agrees on ingress controllers — you need a rewriter for it. External rewriters with the same `name` replace the built-in one. Remember to map them in `dekube.yaml`.

### nginx

| | |
|---|---|
| **Repo** | [dekube-rewriter-nginx](https://github.com/dekubeio/dekube-rewriter-nginx) |
| **Matches** | `nginx` ingressClassName |
| **Status** | stable |

It just translates. Not its fault the controller it serves gets deprecated while the project still can't agree on whether it's called `ingress-nginx`, `nginx-ingress`, or `kubernetes-ingress`. The annotations are a mess. The rewriter faithfully reproduces the mess. Shoot the messenger if you want, but maybe update your ingress controller first.

Handles `rewrite-target`, `backend-protocol`, CORS, `proxy-body-size`, `configuration-snippet`. If your helmfile uses nginx annotations, install this or watch your Caddy routes silently ignore everything that made your app work.

```bash
python3 dekube-manager.py nginx
```

---

### traefik

| | |
|---|---|
| **Repo** | [dekube-rewriter-traefik](https://github.com/dekubeio/dekube-rewriter-traefik) |
| **Matches** | `traefik` ingressClassName |
| **Status** | POC |

A translator that knows it doesn't speak the full language, and chooses silence over hallucination. Traefik CRDs (`IngressRoute`, `Middleware`, etc.) are not supported — only standard `Ingress` resources with Traefik annotations. If your helmfile uses Traefik CRDs extensively, this won't save you. If it uses standard Ingress with a few Traefik-flavored annotations, it might. Handles `router.tls` and standard path rules. Everything else passes through unremarked, unrewritten, unrepentant.

Untested. Known gaps above. Use HAProxy for anything that matters.

```bash
python3 dekube-manager.py traefik
```

## Third-party extensions

Extensions that live outside the dekubeio org. They follow the same contracts and install the same way — but the org takes no credit, no blame, and no responsibility.

### fake-apiserver
A transform that breaches [the wall](understand/concepts.md#the-emulation-boundary). For applications that require a live kube-apiserver at runtime — leader election, service discovery via API, in-cluster auth — this extension provides one. What it does, how it does it, and why you should not use it are documented in the [repo itself](https://github.com/baptisterajaut/dekube-fakeapi). No further instructions will be given here. The catalogue acknowledges its existence; it does not condone it.

## Something not working?

If an extension misbehaves, open an issue on its own repo — each one is linked in the tables above. If you're not sure whether the problem is in the extension or in the engine, use [helmfile2compose](https://github.com/dekubeio/helmfile2compose/issues) — I'll triage.

## Writing your own

- **[Writing converters](extend/extensions/writing-converters.md)** — the generic interface: `kinds`, `convert()`, `ConvertResult`, `ConvertContext`
- **[Writing providers](extend/extensions/writing-providers.md)** — providers: synthetic resources, network alias registration, emulation boundary
- **[Writing transforms](extend/extensions/writing-transforms.md)** — post-processing the final compose output
- **[Writing rewriters](extend/extensions/writing-rewriters.md)** — translating ingress annotations to ingress entries
- **[Writing ingress providers](extend/extensions/writing-ingressproviders.md)** — replacing the reverse proxy backend entirely

Drop it in a `.py` file, and either use `--extensions-dir` locally or publish it as a GitHub repo for dekube-manager distribution. The abyss is open for contributions.
