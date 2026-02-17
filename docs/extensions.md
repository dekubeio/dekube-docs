# Extension catalogue

Extensions are external modules that extend helmfile2compose beyond its built-in capabilities. Install them via [h2c-manager](maintainer/h2c-manager.md) or manually with `--extensions-dir`.

There are two types of extensions: **converters** (teach h2c how to handle new Kubernetes resource kinds) and **transforms** (post-process the final compose output after alias injection). Both are loaded from the same `--extensions-dir`.

CRDs are K8s entities that don't speak compose — exiles from a world with controllers and reconciliation loops. The extension system is the immigration office: each converter forges the documents that the K8s controller would have produced at runtime. The forgery is disturbingly convincing.

## Available extensions

### keycloak

| | |
|---|---|
| **Repo** | [h2c-operator-keycloak](https://github.com/helmfile2compose/h2c-operator-keycloak) |
| **Type** | CRD operator |
| **Kinds** | `Keycloak`, `KeycloakRealmImport` |
| **Dependencies** | none |
| **Priority** | 50 |
| **Status** | stable |

Almost boring by this project's standards. The Keycloak Operator's job is to read a CR and produce a Deployment with the right env vars — which is exactly what h2c does for every other workload anyway. The `Keycloak` CR becomes a compose service with KC_* environment variables (database, HTTP, hostname, proxy, features). `KeycloakRealmImport` CRs are written as JSON files and mounted for auto-import on startup. A realm import is a static declaration that gets applied once — no reconciliation loop needed, no mutation, no drift to watch for. Turns out, removing the operator from a CRD that was already declarative just... works. The least heretical thing here.

Features: TLS secret mounting (if certs exist — doesn't care who made them), podTemplate volume support, bootstrap admin generation, realm placeholder resolution, namespace and K8s Service alias registration for network aliases.

```bash
python3 h2c-manager.py keycloak
```

---

### cert-manager

| | |
|---|---|
| **Repo** | [h2c-operator-cert-manager](https://github.com/helmfile2compose/h2c-operator-cert-manager) |
| **Type** | CRD operator |
| **Kinds** | `Certificate`, `ClusterIssuer`, `Issuer` |
| **Dependencies** | `cryptography` (Python package) |
| **Priority** | 10 |
| **Status** | stable |

The most heretical of all extensions — and paradoxically, the one with the strongest case for existing. Try setting up a local CA chain, issuing certs with the right SANs for a dozen services, and mounting them where they need to go, all by hand in a compose file. cert-manager's declarative model actually makes *more* sense going through h2c than doing it manually. That's the uncomfortable part: the ICBM-to-kill-flies pipeline is, for once, genuinely simpler than the alternative.

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
| **Repo** | [h2c-operator-trust-manager](https://github.com/helmfile2compose/h2c-operator-trust-manager) |
| **Type** | CRD operator |
| **Kinds** | `Bundle` |
| **Dependencies** | `cert-manager` extension; optional `certifi` (falls back to system CA paths) |
| **Priority** | 20 |
| **Status** | stable |

The accomplice. Assembles CA trust bundles from cert-manager Secrets, ConfigMaps, inline PEM, and system default CAs. Injects the result as a synthetic ConfigMap. Pods that mount the trust bundle ConfigMap get the assembled CA chain automatically — believing they live in a cluster where a trust-manager controller reconciled this for them.

Depends on the cert-manager extension (needs its generated secrets). When installed via h2c-manager, cert-manager is auto-resolved as a dependency.

```bash
python3 h2c-manager.py trust-manager
# cert-manager is installed automatically as a dependency
```

---

### servicemonitor

| | |
|---|---|
| **Repo** | [h2c-operator-servicemonitor](https://github.com/helmfile2compose/h2c-operator-servicemonitor) |
| **Type** | CRD operator |
| **Kinds** | `Prometheus`, `ServiceMonitor` |
| **Dependencies** | none |
| **Priority** | 60 |
| **Status** | stable |

Why would you need monitoring in a compose stack? You wouldn't. You absolutely wouldn't. And yet someone asked, and now here we are — a full Prometheus with auto-generated scrape targets, in docker compose, because apparently the shed needs an alarm system.

In Kubernetes, the Prometheus Operator watches ServiceMonitor CRDs and rewrites scrape config dynamically. This extension reads the same CRDs and bakes everything into a static `prometheus.yml`. No operator, no watch, no dynamic anything. Prometheus doesn't know the difference. Prometheus doesn't need to know.

Features: FQDN scrape targets (via network aliases), HTTPS scrape with CA bundle mounting (uses trust-manager ConfigMaps if available), named port resolution, label-based Service matching, fallback name-based matching for operator-created Services (e.g. Keycloak). No hard dependencies on other extensions — works standalone for HTTP scrape targets.

Grafana saw what happened to its lifelong companion and did not intend to suffer the same fate — it [fought back](maintainer/known-workarounds/kube-prometheus-stack.md).

```bash
python3 h2c-manager.py servicemonitor
```

## Transforms

Transforms are extensions that modify the final compose output *after* converters have run. They don't produce services from K8s manifests — they reshape what's already there. Whether this constitutes repair or further damage depends on your perspective.

### flatten-internal-urls

| | |
|---|---|
| **Repo** | [h2c-transform-flatten-internal-urls](https://github.com/helmfile2compose/h2c-transform-flatten-internal-urls) |
| **Type** | Transform |
| **Dependencies** | none |
| **Priority** | 200 |
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

## Writing your own

See [Writing operators](developer/writing-operators.md) and [Writing transforms](developer/writing-transforms.md) for the full guides. The short version:

- **Converter**: a class with `kinds` (list of K8s kinds) and `convert(kind, manifests, ctx)` (returns a `ConvertResult`).
- **Transform**: a class with `transform(compose_services, caddy_entries, ctx)` and no `kinds`. Mutates in place.

Drop it in a `.py` file, and either use `--extensions-dir` locally or publish it as a GitHub repo for h2c-manager distribution. The abyss is open for contributions.
