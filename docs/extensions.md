# Extension catalogue

Extensions are external modules that extend helmfile2compose beyond its built-in capabilities. Install them via [h2c-manager](maintainer/h2c-manager.md) or manually with `--extensions-dir`.

Today, all extensions are **CRD operators** — converters that teach h2c how to handle Kubernetes Custom Resources. But the extension system is not limited to CRDs. Ingress rewriters, custom secret resolvers, alternative ConfigMap handlers — anything that fits the `kinds` + `convert()` interface can be an extension. The gate is open. For the glory of Yog Sa'rath.

CRDs are K8s entities that don't speak compose — exiles from a world with controllers and reconciliation loops. The extension system is the immigration office: each converter teaches h2c a new language, emulating what the K8s controller would have produced at runtime.

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

Converts Keycloak CRDs into compose services. The `Keycloak` CR becomes a compose service with KC_* environment variables (database, HTTP, hostname, proxy, features). `KeycloakRealmImport` CRs are written as JSON files and mounted for auto-import on startup.

Features: TLS secret mounting, podTemplate volume support, bootstrap admin generation, realm placeholder resolution, namespace and K8s Service alias registration for network aliases.

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

Generates real PEM certificates at conversion time — CA chains, SANs, ECDSA/RSA — and injects them as synthetic K8s Secrets into the conversion context. Workloads that mount these Secrets pick them up through the existing volume-mount machinery.

Processes certificates in rounds: self-signed CAs first, then CA-issued certs. Merges duplicate `secretName` entries across namespaces into a single cert with combined SANs.

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
| **Dependencies** | optional `certifi` (falls back to system CA paths) |
| **Priority** | 20 |
| **Status** | stable |

Assembles CA trust bundles from cert-manager Secrets, ConfigMaps, inline PEM, and system default CAs. Injects the result as a synthetic ConfigMap. Pods that mount the trust bundle ConfigMap get the assembled CA chain automatically.

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
| **Dependencies** | `pyyaml` (already required by h2c-core) |
| **Priority** | 60 |
| **Status** | stable |

Converts kube-prometheus-stack CRDs into a Prometheus compose service with auto-generated scrape configuration. The `Prometheus` CR provides image/version/retention. `ServiceMonitor` CRs are resolved against K8s Services to produce `static_configs` in `prometheus.yml`.

Features: FQDN scrape targets (via network aliases), HTTPS scrape with CA bundle mounting (uses trust-manager ConfigMaps if available), named port resolution, label-based Service matching, fallback name-based matching for operator-created Services (e.g. Keycloak). No hard dependencies on other extensions — works standalone for HTTP scrape targets.

For Grafana setup (k8s-sidecar workaround), see [kube-prometheus-stack workaround](maintainer/known-workarounds/kube-prometheus-stack.md).

```bash
python3 h2c-manager.py servicemonitor
```

## Writing your own

See [Writing operators](developer/writing-operators.md) for the full guide. The short version: write a Python class with `kinds` (list of K8s kinds to handle) and `convert(kind, manifests, ctx)` (returns a `ConvertResult`), drop it in a `.py` file, and either use `--extensions-dir` locally or publish it as a GitHub repo for h2c-manager distribution.
