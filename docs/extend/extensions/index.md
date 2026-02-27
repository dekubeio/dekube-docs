# Writing extensions

Before writing an extension, make sure you're familiar with [Concepts](../../understand/concepts.md) (design philosophy, emulation boundary) and [Architecture](../../understand/architecture.md) (converter pipeline, dispatch loop). A look at [Code quality](../code-quality.md) is also recommended — the bar is higher than the project's origins would suggest.

## Extension types

| Type | Interface | Page | Naming convention |
|------|-----------|------|-------------------|
| **Converter** | `kinds` + `convert()` | [Writing converters](writing-converters.md) | `dekube-converter-*` |
| **Provider** | subclass of `Converter`, `kinds` + `convert()` | [Writing providers](writing-providers.md) | `dekube-provider-*` |
| **Ingress provider** | subclass of `IngressProvider` | [Writing ingress providers](writing-ingressproviders.md) | distribution-level |
| **Transform** | `transform()`, no `kinds` | [Writing transforms](writing-transforms.md) | `dekube-transform-*` |
| **Indexer** | subclass of `IndexerConverter`, `kinds` + `convert()` | [Writing converters](writing-converters.md) | `dekube-indexer-*` |
| **Ingress rewriter** | `name` + `match()` + `rewrite()` | [Writing rewriters](writing-rewriters.md) | `dekube-rewriter-*` |

Converters, providers, and indexers share the same code interface — but the distinction is now enforced. `Provider` is a base class in `dekube.pacts.types`; subclassing it signals that the extension produces compose services. `IndexerConverter` is a base class for extensions that populate `ConvertContext` lookups (e.g. `ctx.configmaps`, `ctx.secrets`) without producing output. See the [Extension catalogue](../../catalogue.md) for the full list.

## Available imports

```python
from dekube import ConvertContext          # passed to convert() / rewrite()
from dekube import ConverterResult        # return type for converters/indexers (no services)
from dekube import ProviderResult         # return type for providers (with services)
from dekube import ConvertResult           # deprecated alias for ProviderResult
from dekube import Converter               # base class for converters
from dekube import IndexerConverter        # base class for indexers (populate ctx, no output)
from dekube import Provider                # base class for providers (produce compose services)
from dekube import IngressRewriter         # base class for ingress rewriters
from dekube import get_ingress_class       # resolve ingressClassName + ingress_types
from dekube import resolve_backend         # v1/v1beta1 backend → upstream dict
from dekube import apply_replacements      # apply user-defined string replacements
from dekube import resolve_env             # resolve env/envFrom into flat list
from dekube import _secret_value           # decode a Secret key (base64 or plain)
```

These are the **pacts** — the [sacred contracts](../../understand/engine.md#pacts--the-sacred-contracts) — and are stable across minor versions. Both import paths work:

```python
from dekube import ConvertContext           # via re-export
from dekube.pacts import ConvertContext     # explicit
```

- **`ConverterResult`** — return type for converters and indexers. One field: `ingress_entries` (list). Use when your extension doesn't produce compose services.
- **`ProviderResult`** — return type for providers. Two fields: `services` (dict) and `ingress_entries` (list, inherited from `ConverterResult`).
- **`ConvertResult`** — deprecated alias for `ProviderResult`. Still works, but prefer the typed variants.
- **`Converter`** — base class for all converters (default priority 1000). Optional — duck typing works, but subclassing provides defaults.
- **`IndexerConverter`** — base class for indexers that populate `ConvertContext` lookups (e.g. `ctx.configmaps`, `ctx.secrets`) without producing output. Default priority 50.
- **`Provider`** — base class for converters that produce compose services (default priority 500). CRD extensions that return `ProviderResult` with non-empty `services` should subclass this. See [Writing providers](writing-providers.md#provider-vs-converter).
- **`IngressRewriter`** — base class for ingress rewriters. Subclass it or implement the same duck-typed contract.
- **`get_ingress_class(manifest, ingress_types)`** — resolves `ingressClassName` from spec or annotation, then through the `ingress_types` config mapping.
- **`resolve_backend(path_entry, manifest, ctx)`** — resolves a v1/v1beta1 Ingress backend to `{svc_name, compose_name, container_port, upstream, ns}`.
- **`apply_replacements(text, replacements)`** — applies user-defined `replacements` (from `ctx.replacements`) to a string.
- **`resolve_env(container, configmaps, secrets, workload_name, warnings, replacements=None, service_port_map=None)`** — resolves a container's `env` and `envFrom` into a flat `list[dict]` of `{name, value}` pairs.
- **`_secret_value(secret, key)`** — decodes a single key from a K8s Secret dict. Handles both `stringData` (plain text) and `data` (base64-decoded). Returns `str | None`. Useful for converters that need to read Secret values injected by other converters (e.g. reading a database password from a cert-manager-generated secret). The underscore prefix is historical — the function is part of the stable pacts API.

The other `_`-prefixed functions (`_apply_port_remap`, `_apply_alias_map`, etc.) still exist but may change between versions. Pin your dekube-engine version if you depend on them. Transforms in particular should avoid importing from the core — see [Writing transforms](writing-transforms.md#self-contained--no-core-imports).

## Testing locally

All extension types are loaded from the same `--extensions-dir`. The loader detects each type automatically — converters, transforms, and rewriters can coexist in the same directory.

Testing with the **bare core** (`dekube.py`) — the core has no built-in converters, so you'll only see your extension's output:

```bash
python3 dekube.py --from-dir /tmp/rendered \
  --extensions-dir ./my-extensions --output-dir ./output
```

Testing with the **distribution** (`helmfile2compose.py`) — includes all built-in converters, so you see full output:

```bash
python3 helmfile2compose.py --from-dir /tmp/rendered \
  --extensions-dir ./my-extensions --output-dir ./output
```

Check the output for load confirmation:

```
Loaded extensions: MyConverter (MyCustomResource)
Loaded transforms: MyTransform
Loaded rewriters: NginxRewriter (nginx)
```

## Repo structure

For distribution via [dekube-manager](https://manager.dekube.io/docs/), each extension is a GitHub repo with:

```
dekube-{type}-{name}/
├── {name}.py              # extension class (mandatory)
├── requirements.txt       # Python deps, if any (optional)
└── README.md              # description, kinds/purpose, usage (mandatory)
```

The `.py` file must be in the repo root. `requirements.txt` follows pip format — dekube-manager checks if deps are installed and warns if not.

The README should cover: what the extension does, handled kinds (for converters/providers), dependencies, priority, usage example.

## Publishing

1. Create a GitHub repo under the `dekubeio` org (or your own account).
2. Create a GitHub Release with a tag (e.g. `v0.1.0`). The release doesn't need assets — the tag is what matters.
3. Open a PR to `dekubeio/dekube-manager` adding your extension to `extensions.json`:

```json
{
  "schema_version": 1,
  "extensions": {
    "my-extension": {
      "repo": "dekubeio/dekube-{type}-{name}",
      "description": "What it does",
      "file": "{name}.py",
      "depends": [],
      "incompatible": []
    }
  }
}
```

`depends` lists extensions that must be installed alongside. `incompatible` lists extensions that conflict (bidirectional — declaring on one side is enough). Both are optional.

Once merged, users can install with:

```bash
python3 dekube-manager.py my-extension
```
