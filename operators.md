# Operator catalogue

Operators are external CRD converters that extend helmfile2compose to handle Kubernetes Custom Resources. Install them via [h2c-manager](maintainer/h2c-manager.md) or manually with `--extensions-dir`.

CRDs are K8s entities that don't speak compose — exiles from a world with controllers and reconciliation loops. The operator system is the immigration office: each converter teaches h2c a new language, emulating what the K8s controller would have produced at runtime.

## Available operators

### keycloak

| | |
|---|---|
| **Repo** | [h2c-operator-keycloak](https://github.com/helmfile2compose/h2c-operator-keycloak) |
| **Kinds** | `Keycloak`, `KeycloakRealmImport` |
| **Dependencies** | none |
| **Priority** | 50 |
| **Status** | stable |

Converts Keycloak operator CRDs into compose services. The `Keycloak` CR becomes a compose service with KC_* environment variables (database, HTTP, hostname, proxy, features). `KeycloakRealmImport` CRs are written as JSON files and mounted for auto-import on startup.

Features: TLS secret mounting, podTemplate volume support, bootstrap admin generation, realm placeholder resolution, K8s DNS rewriting in realm data.

```bash
python3 h2c-manager.py keycloak
```

---

### certmanager

| | |
|---|---|
| **Repo** | [h2c-operator-certmanager](https://github.com/helmfile2compose/h2c-operator-certmanager) |
| **Kinds** | `Certificate`, `ClusterIssuer`, `Issuer` |
| **Dependencies** | `cryptography` (Python package) |
| **Priority** | 10 |
| **Status** | stable |

Generates real PEM certificates at conversion time — CA chains, SANs, ECDSA/RSA — and injects them as synthetic K8s Secrets into the conversion context. Workloads that mount these Secrets pick them up through the existing volume-mount machinery.

Processes certificates in rounds: self-signed CAs first, then CA-issued certs. Merges duplicate `secretName` entries across namespaces into a single cert with combined SANs.

```bash
python3 h2c-manager.py certmanager
pip install cryptography  # required dependency
```

---

### trust-manager

| | |
|---|---|
| **Repo** | [h2c-operator-trust-manager](https://github.com/helmfile2compose/h2c-operator-trust-manager) |
| **Kinds** | `Bundle` |
| **Dependencies** | optional `certifi` (falls back to system CA paths) |
| **Priority** | 20 |
| **Status** | stable |

Assembles CA trust bundles from cert-manager Secrets, ConfigMaps, inline PEM, and system default CAs. Injects the result as a synthetic ConfigMap. Pods that mount the trust bundle ConfigMap get the assembled CA chain automatically.

Depends on the certmanager operator (needs its generated secrets). When installed via h2c-manager, certmanager is auto-resolved as a dependency.

```bash
python3 h2c-manager.py trust-manager
# certmanager is installed automatically as a dependency
```

## Writing your own

See [Writing operators](developer/writing-operators.md) for the full guide. The short version: write a Python class with `kinds` (list of K8s kinds to handle) and `convert(kind, manifests, ctx)` (returns a `ConvertResult`), drop it in a `.py` file, and either use `--extensions-dir` locally or publish it as a GitHub repo for h2c-manager distribution.
