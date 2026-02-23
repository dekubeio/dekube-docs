# Extensions

Extensions are external modules that extend helmfile2compose beyond its built-in capabilities. Install them via [h2c-manager](maintainer/h2c-manager.md) or manually with `--extensions-dir`.

CRDs are K8s entities that don't speak compose — exiles from a world with controllers and reconciliation loops. The extension system is the immigration office: each converter forges the documents that the K8s controller would have produced at runtime. The forgery is disturbingly convincing.

## Extension types

There are five extension types, all loaded from the same `--extensions-dir`:

| Type | Interface | Purpose | Naming convention |
|------|-----------|---------|-------------------|
| **IndexerConverter** | `kinds` + `convert()` | Populate `ConvertContext` lookups (configmaps, secrets, PVCs, services) without producing services | `h2c-indexer-*` |
| **Converter** | `kinds` + `convert()` | Handle K8s resource kinds, produce synthetic resources (Secrets, ConfigMaps) | `h2c-converter-*` |
| **Provider** | `kinds` + `convert()` | Handle K8s resource kinds, produce compose services (and possibly resources) | `h2c-provider-*` |
| **Transform** | `transform()`, no `kinds` | Post-process the final compose output after converters | `h2c-transform-*` |
| **Ingress rewriter** | `name` + `match()` + `rewrite()` | Translate ingress controller annotations into ingress entries | `h2c-rewriter-*` |

All converters share the same code interface (`kinds` + `convert()` → `ConverterResult`/`ProviderResult`), but the repo naming convention signals what they produce:

- **`h2c-converter-*`** — produces synthetic resources (Secrets, ConfigMaps, files on disk) without adding compose services. The extension *converts* K8s resources into other resources. Examples: cert-manager generates PEM certificates as Secrets, trust-manager assembles CA bundles as ConfigMaps.
- **`h2c-provider-*`** — produces compose services (and possibly resources too). The extension *provides* running containers that emulate what a K8s controller would have created. Examples: keycloak turns a CR into a compose service, servicemonitor produces a Prometheus service with baked-in scrape config.

The distinction is enforced: `Provider` is a base class in `h2c.pacts.types`. Subclassing it signals that the extension produces compose services.

See [Writing converters](developer/extensions/writing-converters.md) for the generic interface. For providers (CRD extensions that produce compose services), see [Writing providers](developer/extensions/writing-providers.md). For reverse proxy providers, see [Writing ingress providers](developer/extensions/writing-ingressproviders.md).

## Providers

Providers produce compose services — they emulate what a K8s controller would have created as running workloads.

### keycloak

| | |
|---|---|
| **Repo** | [h2c-provider-keycloak](https://github.com/helmfile2compose/h2c-provider-keycloak) |
| **Kinds** | `Keycloak`, `KeycloakRealmImport` |
| **Dependencies** | none |
| **Priority** | 500 |
| **Produces** | compose services + realm JSON files |
| **Status** | stable |

Almost boring by this project's standards. The Keycloak Operator's job is to read a CR and produce a Deployment with the right env vars — which is exactly what h2c does for every other workload anyway. The `Keycloak` CR becomes a compose service with KC_* environment variables (database, HTTP, hostname, proxy, features). `KeycloakRealmImport` CRs are written as JSON files and mounted for auto-import on startup. A realm import is a static declaration that gets applied once — no reconciliation loop needed, no mutation, no drift to watch for. Turns out, removing the operator from a CRD that was already declarative just... works. Barely heretical.

Features: TLS secret mounting (if certs exist — doesn't care who made them), podTemplate volume support, bootstrap admin generation, realm placeholder resolution, namespace and K8s Service alias registration for network aliases.

```bash
python3 h2c-manager.py keycloak
```

---

### servicemonitor

| | |
|---|---|
| **Repo** | [h2c-provider-servicemonitor](https://github.com/helmfile2compose/h2c-provider-servicemonitor) |
| **Kinds** | `Prometheus`, `ServiceMonitor` |
| **Dependencies** | none |
| **Priority** | 600 |
| **Produces** | compose services + prometheus.yml |
| **Status** | stable |

Why would you need monitoring in a compose stack? You wouldn't. You absolutely wouldn't. And yet someone asked, and now here we are — a full Prometheus with auto-generated scrape targets, in docker compose, because apparently the shed needs an alarm system.

In Kubernetes, the Prometheus Operator watches ServiceMonitor CRDs and rewrites scrape config dynamically. This extension reads the same CRDs and bakes everything into a static `prometheus.yml`. No operator, no watch, no dynamic anything. Prometheus doesn't know the difference. Prometheus doesn't need to know.

Features: FQDN scrape targets (via network aliases), HTTPS scrape with CA bundle mounting (uses trust-manager ConfigMaps if available), named port resolution, label-based Service matching, fallback name-based matching for converter-created resources (e.g. Keycloak). No hard dependencies on other extensions — works standalone for HTTP scrape targets.

```bash
python3 h2c-manager.py servicemonitor
```

Grafana saw what we did to its lifelong companion and [fought back itself](maintainer/known-workarounds/kube-prometheus-stack.md).

## Converters

Converters produce synthetic resources (Secrets, ConfigMaps, files on disk) without adding compose services. They forge the documents that a K8s controller would have produced at runtime — the resources exist, but no workload runs.

### cert-manager

| | |
|---|---|
| **Repo** | [h2c-converter-cert-manager](https://github.com/helmfile2compose/h2c-converter-cert-manager) |
| **Kinds** | `Certificate`, `ClusterIssuer`, `Issuer` |
| **Dependencies** | `cryptography` (Python package) |
| **Priority** | 100 |
| **Produces** | synthetic Secrets (PEM certificates) |
| **Status** | stable |

Very heretical — and paradoxically, the one with a strong case for existing. Try setting up a local CA chain, issuing certs with the right SANs for a dozen services, and mounting them where they need to go, all by hand in a compose file. cert-manager's declarative model actually makes *more* sense going through helmfile2compose than doing it manually. That's the uncomfortable part: the ICBM-to-kill-flies pipeline is, for once, genuinely simpler than the alternative.

This extension generates real PEM certificates at conversion time — CA chains, SANs, ECDSA/RSA — and injects them as synthetic K8s Secrets into the conversion context. No ACME challenges. No renewal. No controller. Just cryptographic material, conjured from nothing, valid until it isn't.

Then it starts merging certificates. Duplicate `secretName` entries across namespaces? Combined into a single cert with merged SANs. Rounds of issuance — self-signed CAs first, then CA-issued certs — because dependency order matters even in forgery. The kind of extension you can't predict, can't control, and can't entirely disapprove of — because the results speak for themselves, even if the methods are grounds for intervention.

```bash
python3 h2c-manager.py cert-manager
pip install cryptography  # required dependency
```

---

### trust-manager

| | |
|---|---|
| **Repo** | [h2c-converter-trust-manager](https://github.com/helmfile2compose/h2c-converter-trust-manager) |
| **Kinds** | `Bundle` |
| **Dependencies** | `cert-manager` extension; optional `certifi` (falls back to system CA paths) |
| **Priority** | 200 |
| **Produces** | synthetic ConfigMaps (CA bundles) |
| **Status** | stable |

The accomplice. Assembles CA trust bundles from cert-manager Secrets, ConfigMaps, inline PEM, and system default CAs. Injects the result as a synthetic ConfigMap. Pods that mount the trust bundle ConfigMap get the assembled CA chain automatically — believing they live in a cluster where a trust-manager controller reconciled this for them.

Depends on the cert-manager extension (needs its generated secrets). When installed via h2c-manager, cert-manager is auto-resolved as a dependency.

```bash
python3 h2c-manager.py trust-manager
# cert-manager is installed automatically as a dependency
```

## Transforms

Transforms are extensions that modify the final compose output *after* converters have run. They don't produce services from K8s manifests — they reshape what's already there. Whether this constitutes repair or further damage depends on your perspective.

### flatten-internal-urls

| | |
|---|---|
| **Repo** | [h2c-transform-flatten-internal-urls](https://github.com/helmfile2compose/h2c-transform-flatten-internal-urls) |
| **Dependencies** | none |
| **Priority** | 2000 |
| **Incompatible with** | `cert-manager` |
| **Status** | stable |

The only extension with a heresy score of NaN/10. It destroys the K8s naming temple — but in doing so, it reduces the overall desecration. Whether this makes it a sin or a penance is left as an exercise for the theologian.

Strips Docker Compose network aliases and rewrites K8s FQDNs (`svc.ns.svc.cluster.local`) to short compose service names. Rewrites env vars, configmap files on disk, and Caddy upstreams. v2.1 spent considerable effort building the alias system so that K8s names would carry into compose; this transform rips it all out. On purpose.

Built to restore **nerdctl compose** compatibility — nerdctl silently ignores network aliases, so FQDNs never resolve. But nerdctl isn't the only reason to use it. The transform also produces **cleaner compose output** — no 4-line alias blocks on every service, no `keycloak.auth.svc.cluster.local` in environment variables when `keycloak` would do. If you don't need FQDN preservation (no inter-service TLS, no cert SANs to match), flattening makes the generated files easier to read, debug, and diff.

**Incompatible with cert-manager**: the cert-manager extension generates certificates with SANs matching K8s FQDNs. Flattening those FQDNs breaks TLS verification. h2c-manager blocks this combination by default — use `--ignore-compatibility-errors` if you know what you're doing.

```bash
python3 h2c-manager.py flatten-internal-urls
```

---

### bitnami

| | |
|---|---|
| **Repo** | [h2c-transform-bitnami](https://github.com/helmfile2compose/h2c-transform-bitnami) |
| **Dependencies** | none |
| **Priority** | 1500 |
| **Status** | stable |

The janitor. Bitnami charts — Redis, PostgreSQL, Keycloak — wrap standard images in custom entrypoints, init containers, and volume conventions that assume a full Kubernetes environment. In compose, the entrypoints fail, the volumes don't line up, and the init containers crash on missing emptyDirs. The workarounds are well-documented in [common charts](maintainer/known-workarounds/common-charts.md) — this transform applies them automatically so you don't have to copy-paste overrides across projects.

Detects Bitnami images by name, then: replaces Redis entirely with stock `redis:7-alpine`, fixes PostgreSQL volume paths, injects Keycloak passwords as env vars and removes the failing init container. Every modification is logged to stderr. User `overrides:` take precedence — if you've already handled a service manually, the transform leaves it alone.

```bash
python3 h2c-manager.py bitnami
```

## Ingress rewriters

Ingress rewriters translate controller-specific annotations into Caddy configuration. Unlike converters, they don't claim K8s kinds — they intercept individual Ingress manifests based on `ingressClassName` or annotation prefix, read whatever vendor-specific incantations the chart author scattered across the annotations, and produce Caddy config that does the same thing. The annotations were never meant to be portable. That's the point.

The built-in `HAProxyRewriter` handles `haproxy` and empty/absent ingress classes. If your cluster uses something else — and statistically, it does, because nobody agrees on ingress controllers — you need a rewriter for it. External rewriters with the same `name` replace the built-in one. Remember to map them in `helmfile2compose.yaml`.

### nginx

| | |
|---|---|
| **Repo** | [h2c-rewriter-nginx](https://github.com/helmfile2compose/h2c-rewriter-nginx) |
| **Matches** | `nginx` ingressClassName |
| **Status** | stable |

It just translates. Not its fault the controller it serves gets deprecated while the project still can't agree on whether it's called `ingress-nginx`, `nginx-ingress`, or `kubernetes-ingress`. The annotations are a mess. The rewriter faithfully reproduces the mess. Shoot the messenger if you want, but maybe update your ingress controller first.

Handles `rewrite-target`, `backend-protocol`, CORS, `proxy-body-size`, `configuration-snippet`. If your helmfile uses nginx annotations, install this or watch your Caddy routes silently ignore everything that made your app work.

```bash
python3 h2c-manager.py nginx
```

---

### traefik

| | |
|---|---|
| **Repo** | [h2c-rewriter-traefik](https://github.com/helmfile2compose/h2c-rewriter-traefik) |
| **Matches** | `traefik` ingressClassName |
| **Status** | POC |

A translator that knows it doesn't speak the full language, and chooses silence over hallucination. Traefik CRDs (`IngressRoute`, `Middleware`, etc.) are not supported — only standard `Ingress` resources with Traefik annotations. If your helmfile uses Traefik CRDs extensively, this won't save you. If it uses standard Ingress with a few Traefik-flavored annotations, it might. Handles `router.tls` and standard path rules. Everything else passes through unremarked, unrewritten, unrepentant.

Warning: untested. May or may not work. Can't tell. Use HAProxy.

```bash
python3 h2c-manager.py traefik
```

## Ingress providers

Ingress providers are a special kind of provider — they handle Ingress manifests and produce a reverse proxy service + config file. Unlike rewriters (which translate annotations), ingress providers own the entire reverse proxy lifecycle: service creation, config generation, rewriter dispatch.

### caddy (built-in)

| | |
|---|---|
| **Repo** | [h2c-provider-caddy](https://github.com/helmfile2compose/h2c-provider-caddy) |
| **Type** | distribution built-in |
| **Kinds** | `Ingress` |
| **Priority** | 900 |
| **Produces** | Caddy compose service + Caddyfile |
| **Status** | stable |

The default ingress provider bundled with the helmfile2compose distribution. Dispatches each Ingress manifest to the first matching rewriter, collects entries, builds a Caddy service with CA cert mounts, and writes a Caddyfile with host blocks. Supports `disable_ingress` mode, `tls internal`, backend SSL via trust pools, `extra_directives`, and `strip_prefix`.

This is the reference implementation for `IngressProvider`. See [Writing ingress providers](developer/extensions/writing-ingressproviders.md) to build your own.

## Writing your own

- **[Writing converters](developer/extensions/writing-converters.md)** — the generic interface: `kinds`, `convert()`, `ConvertResult`, `ConvertContext`
- **[Writing providers](developer/extensions/writing-providers.md)** — providers: synthetic resources, network alias registration, emulation boundary
- **[Writing transforms](developer/extensions/writing-transforms.md)** — post-processing the final compose output
- **[Writing rewriters](developer/extensions/writing-rewriters.md)** — translating ingress annotations to ingress entries
- **[Writing ingress providers](developer/extensions/writing-ingressproviders.md)** — replacing the reverse proxy backend entirely

Drop it in a `.py` file, and either use `--extensions-dir` locally or publish it as a GitHub repo for h2c-manager distribution. The abyss is open for contributions.
