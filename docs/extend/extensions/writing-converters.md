# Writing converters

A converter teaches dekube how to handle Kubernetes resource kinds it would otherwise skip with a warning and a shrug. You claim a `kind`, you get its manifests, you decide what they become in compose. One `.py` file, one class, one more thing the engine shouldn't know how to do but now does.

> *To name a thing is to summon it. To teach another its name is to bind your fate to what answers.*
>
> — *Book of Eibon, On Names and Bindings (probably²)*

If your converter targets CRD kinds (replacing a K8s controller), see [Writing providers](writing-providers.md) for additional patterns (synthetic resources, network alias registration, cross-converter dependencies). For Ingress-specific reverse proxy backends, see [Writing ingress providers](writing-ingressproviders.md).

The distinction between converters (`dekube-converter-*`) and providers (`dekube-provider-*`) is not a naming suggestion — it's enforced. `Provider` is a base class in `dekube.pacts.types`, and subclassing it tells the engine you intend to produce compose services. Get this wrong and your services vanish without a trace. Both use typed return contracts (`ConverterResult` / `ProviderResult`), and the framework trusts the contract more than your intentions.

!!! warning "Services from non-Provider converters are silently dropped"
    If your converter returns `services` without subclassing `Provider`, those services are silently discarded. No error. No warning. They just vanish from the output, and you spend an hour wondering why your compose file is empty. Subclass `Provider` and return `ProviderResult`. Non-provider converters should return `ConverterResult` (which has no `services` field). The typed contracts enforce this — and the engine enforces the contracts with the quiet cruelty of a system that was never designed to explain itself.

## The contract

A converter class must have:

1. **`kinds`** — a list of K8s kinds to handle (e.g. `["Keycloak", "KeycloakRealmImport"]`). **Kinds are exclusive between extensions** — if two extensions claim the same kind, dekube exits with an error. An extension *can* override a built-in converter by claiming the same kind — the built-in is silently removed from the dispatch for that kind. Yes, this means you can replace how dekube handles Secrets, or Deployments. Why you would corrupt the already corrupted is between you and Yog Sa'rath. (For Ingress annotation handling, use an [ingress rewriter](writing-rewriters.md) instead — converters handle the kind dispatch, rewriters handle the annotation translation.)
2. **`convert(kind, manifests, ctx)`** — called once per kind, returns a `ConvertResult`

```python
from dekube import Converter, ConverterResult

class MyConverter(Converter):
    kinds = ["MyCustomResource"]
    priority = 100

    def convert(self, kind, manifests, ctx):
        for m in manifests:
            name = m.get("metadata", {}).get("name", "?")
            spec = m.get("spec") or {}
            # Inject a synthetic Secret for downstream converters
            ctx.secrets[f"{name}-credentials"] = {
                "metadata": {"name": f"{name}-credentials"},
                "stringData": {"password": spec.get("password", "changeme")},
            }
        return ConverterResult()
```

If your converter produces compose services, subclass `Provider` instead and return `ProviderResult` — see [Writing providers](writing-providers.md).

### Return types

What you return determines what the engine does with your output — and what it quietly throws away. Two return types:

- **`ConverterResult`** — one field: `ingress_entries` (list of dicts). For converters and indexers that don't produce services. Default: empty list.
- **`ProviderResult`** — two fields: `services` (dict) and `ingress_entries` (list, inherited). For providers that produce compose services. Both default to empty.
- **`ConvertResult`** — deprecated alias for `ProviderResult`. Still accepted.

Each ingress entry has `host`, `path`, `upstream`, `scheme`, and optional `server_ca_secret`, `server_sni`, `strip_prefix`, `extra_directives`. Ingress rewriters are the primary producers; converters rarely need to produce them directly.

Most converters return `ConverterResult()` (empty). Providers return `ProviderResult(services=services)`.

### `ConvertContext` (`ctx`)

The shared mutable state that every converter reads from — and writes to. This is how converters communicate: not through return values, but through side effects on a shared dict. Kubernetes had an API server for this. We have a Python object passed by reference. Same energy, fewer HTTP calls.

| Attribute | Type | Description |
|-----------|------|-------------|
| `ctx.configmaps` | `dict` | Indexed ConfigMaps (name -> manifest). **Writable** — converters can inject synthetic ConfigMaps (see [Writing providers](writing-providers.md)). |
| `ctx.secrets` | `dict` | Indexed Secrets (name -> manifest). **Writable** — converters can inject synthetic secrets (see [Writing providers](writing-providers.md)). |
| `ctx.config` | `dict` | The `dekube.yaml` config |
| `ctx.output_dir` | `str` | Output directory for generated files |
| `ctx.warnings` | `list[str]` | Append warnings here (printed to stderr) |
| `ctx.generated_cms` | `set[str]` | Names of ConfigMaps already written to disk |
| `ctx.generated_secrets` | `set[str]` | Names of Secrets already written to disk |
| `ctx.replacements` | `list[dict]` | User-defined string replacements |
| `ctx.alias_map` | `dict` | Service alias map (K8s Service name -> workload name) |
| `ctx.service_port_map` | `dict` | Service port map ((svc_name, port) -> container_port) |
| `ctx.fix_permissions` | `dict[str, int]` | Legacy field (kept for backwards compatibility). Previously used to track PVC claim → UID mappings for permission fixing. The built-in [fix-permissions](https://github.com/dekubeio/dekube-transform-fix-permissions) transform now handles this by scanning K8s manifests and final compose volumes directly. |
| `ctx.services_by_selector` | `dict` | Index of K8s Services by name. Each entry has `name`, `namespace`, `selector`, `type`, `ports`. Used to resolve Services to compose names, generate network aliases, and build port maps. **Writable** — converters should register runtime-created Services here. |
| `ctx.pvc_names` | `set[str]` | Names of PersistentVolumeClaims discovered in manifests. Used to distinguish PVC mounts from other volume types during conversion. |
| `ctx.manifests` | `dict[str, list]` | All parsed K8s manifests, keyed by kind (e.g. `{"Deployment": [...], "Service": [...]}`). Read-only — useful for transforms or converters that need to inspect manifests outside their own `kinds`. |
| `ctx.first_run` | `bool` | `True` if `dekube.yaml` didn't exist before this run. Used to gate auto-population of volumes and excludes. |
| `ctx.extension_config` | `dict` | Per-converter config section from `dekube.yaml`. Set automatically before each `convert()` call, keyed by the converter's `name` attribute. Empty dict if not configured. Configured in `dekube.yaml` under `extensions.<name>`: |

```yaml
# dekube.yaml
extensions:
  my-extension:
    my_key: my_value    # → ctx.extension_config["my_key"]
    enabled: false       # → extension is loaded but never called
```

See [Configuration reference](../../reference/config.md#per-extension-config-extensions) for the full schema.

### Priority

Set `priority` as a class attribute to control execution order. Lower = earlier. Default: 1000 for `Converter`, 50 for `IndexerConverter`, 500 for `Provider`.

```python
class CertManagerConverter:
    kinds = ["Certificate", "ClusterIssuer", "Issuer"]
    priority = 10  # runs first
```

This matters when converters depend on each other's output — which is temporal coupling in a system that was supposed to be "just extensions running independently." cert-manager must inject its secrets before trust-manager reads them. Get the priority wrong and trust-manager finds an empty `ctx.secrets`, produces an empty CA bundle, and every downstream TLS connection fails for reasons that have nothing to do with TLS.

### Multi-kind dispatch

If your converter handles multiple kinds and needs to process them in order, use the `kind` argument. `convert()` is called once per kind, in the order they appear in the `kinds` list:

```python
class MyConverter:
    kinds = ["DependencyKind", "MainKind"]  # order = call order

    def __init__(self):
        self._indexed = {}

    def convert(self, kind, manifests, ctx):
        if kind == "DependencyKind":
            # Index first, produce nothing yet
            for m in manifests:
                name = m.get("metadata", {}).get("name", "")
                self._indexed[name] = m.get("spec", {})
            return ConverterResult()
        # kind == "MainKind" — use indexed data
        return self._process_main(manifests, ctx)
```

## Indexers {#indexers}

An indexer is a converter that populates `ConvertContext` lookups without producing compose services. The four bundled indexers (configmap, secret, pvc, service) handle standard K8s kinds, but you can write your own for any kind you need indexed.

Subclass `IndexerConverter` instead of `Converter` — the only difference is the default priority (50 instead of 1000), which ensures indexers run before converters and providers that depend on their data.

```python
from dekube import IndexerConverter, ConverterResult

class MyResourceIndexer(IndexerConverter):
    kinds = ["MyResource"]
    name = "my-resource-indexer"

    def convert(self, kind, manifests, ctx):
        for m in manifests:
            name = (m.get("metadata") or {}).get("name", "")
            spec = m.get("spec") or {}
            # Populate ctx so downstream converters/providers can use this data
            ctx.configmaps[f"{name}-generated"] = {
                "data": spec.get("config") or {},
            }
        return ConverterResult()
```

Same contract as converters — `kinds`, `convert()`, `ConverterResult`. The distinction is semantic: indexers feed `ctx`, converters transform resources.

See [Writing extensions](index.md) for testing, repo structure, publishing, and available imports.
