# Writing converters

A converter is an extension that teaches h2c how to handle specific Kubernetes resource kinds. Each converter is a single `.py` file with a converter class.

> *To name a thing is to summon it. To teach another its name is to bind your fate to what answers.*
>
> — *Book of Eibon, On Names and Bindings (probably²)*

If your converter targets CRD kinds (replacing a K8s controller), see [Writing providers](writing-providers.md) for additional patterns (synthetic resources, network alias registration, cross-converter dependencies). For Ingress-specific reverse proxy backends, see [Writing ingress providers](writing-ingressproviders.md).

Note: the distinction between converters (`h2c-converter-*`) and providers (`h2c-provider-*`) is enforced — `Provider` is a base class in `h2c.pacts.types`. Providers produce compose services; converters produce synthetic resources. Both use typed return contracts (`ConverterResult` / `ProviderResult`), but subclassing `Provider` signals your intent to the framework.

!!! warning "Services from non-Provider converters are discarded"
    The dispatch loop ignores `services` returned by converters that don't subclass `Provider`. If your converter produces compose services, subclass `Provider` and return `ProviderResult`. Non-provider converters should return `ConverterResult` (which has no `services` field). The typed contracts enforce this at the API level.

## The contract

A converter class must have:

1. **`kinds`** — a list of K8s kinds to handle (e.g. `["Keycloak", "KeycloakRealmImport"]`). **Kinds are exclusive between extensions** — if two extensions claim the same kind, h2c exits with an error. An extension *can* override a built-in converter by claiming the same kind — the built-in is silently removed from the dispatch for that kind. Yes, this means you can replace how h2c handles Secrets, or Deployments. Why you would corrupt the already corrupted is between you and Yog Sa'rath. (For Ingress annotation handling, use an [ingress rewriter](writing-rewriters.md) instead — converters handle the kind dispatch, rewriters handle the annotation translation.)
2. **`convert(kind, manifests, ctx)`** — called once per kind, returns a `ConvertResult`

```python
from h2c import ProviderResult, Provider

class MyConverter(Provider):
    kinds = ["MyCustomResource"]

    def convert(self, kind, manifests, ctx):
        services = {}
        for m in manifests:
            name = m.get("metadata", {}).get("name", "?")
            spec = m.get("spec", {})
            services[name] = {
                "image": spec.get("image", "mydefault:latest"),
                "restart": "always",
                "environment": {"MY_VAR": spec.get("myField", "default")},
            }
        return ProviderResult(services=services)
```

### Return types

Two return types, depending on whether your converter produces compose services:

- **`ConverterResult`** — one field: `ingress_entries` (list of dicts). For converters and indexers that don't produce services. Default: empty list.
- **`ProviderResult`** — two fields: `services` (dict) and `ingress_entries` (list, inherited). For providers that produce compose services. Both default to empty.
- **`ConvertResult`** — deprecated alias for `ProviderResult`. Still accepted.

Each ingress entry has `host`, `path`, `upstream`, `scheme`, and optional `server_ca_secret`, `server_sni`, `strip_prefix`, `extra_directives`. Ingress rewriters are the primary producers; converters rarely need to produce them directly.

Most converters return `ConverterResult()` (empty). Providers return `ProviderResult(services=services)`.

### `ConvertContext` (`ctx`)

The conversion context passed to every converter. Key attributes:

| Attribute | Type | Description |
|-----------|------|-------------|
| `ctx.configmaps` | `dict` | Indexed ConfigMaps (name -> manifest). **Writable** — converters can inject synthetic ConfigMaps (see [Writing providers](writing-providers.md)). |
| `ctx.secrets` | `dict` | Indexed Secrets (name -> manifest). **Writable** — converters can inject synthetic secrets (see [Writing providers](writing-providers.md)). |
| `ctx.config` | `dict` | The `helmfile2compose.yaml` config |
| `ctx.output_dir` | `str` | Output directory for generated files |
| `ctx.warnings` | `list[str]` | Append warnings here (printed to stderr) |
| `ctx.generated_cms` | `set[str]` | Names of ConfigMaps already written to disk |
| `ctx.generated_secrets` | `set[str]` | Names of Secrets already written to disk |
| `ctx.replacements` | `list[dict]` | User-defined string replacements |
| `ctx.alias_map` | `dict` | Service alias map (K8s Service name -> workload name) |
| `ctx.service_port_map` | `dict` | Service port map ((svc_name, port) -> container_port) |
| `ctx.fix_permissions` | `dict[str, int]` | Legacy field (kept for backwards compatibility). The built-in [fix-permissions](https://github.com/helmfile2compose/h2c-transform-fix-permissions) transform now handles permission fixing by scanning K8s manifests and final compose volumes directly. |
| `ctx.services_by_selector` | `dict` | Index of K8s Services by name. Each entry has `name`, `namespace`, `selector`, `type`, `ports`. Used to resolve Services to compose names, generate network aliases, and build port maps. **Writable** — converters should register runtime-created Services here. |
| `ctx.pvc_names` | `set[str]` | Names of PersistentVolumeClaims discovered in manifests. Used to distinguish PVC mounts from other volume types during conversion. |
| `ctx.extension_config` | `dict` | Per-converter config section from `helmfile2compose.yaml`. Set automatically before each `convert()` call, keyed by the converter's `name` attribute (e.g. `caddy` → `extensions.caddy` in config). Empty dict if not configured. |

### Priority

Set `priority` as a class attribute to control execution order. Lower = earlier. Default: 1000 for `Converter`, 50 for `IndexerConverter`, 500 for `Provider`.

```python
class CertManagerConverter:
    kinds = ["Certificate", "ClusterIssuer", "Issuer"]
    priority = 10  # runs first
```

This matters when converters depend on each other's output (e.g. trust-manager needs cert-manager's generated secrets).

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

See [Writing extensions](index.md) for testing, repo structure, publishing, and available imports.
