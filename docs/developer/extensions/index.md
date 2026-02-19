# Writing extensions

Before writing an extension, make sure you're familiar with [Concepts](../concepts.md) (design philosophy, emulation boundary) and [Architecture](../architecture.md) (converter pipeline, dispatch loop). A look at [Code quality](../code-quality.md) is also recommended — the bar is higher than the project's origins would suggest.

## Extension types

| Type | Interface | Page | Naming convention |
|------|-----------|------|-------------------|
| **Converter** | `kinds` + `convert()` | [Writing converters](writing-converters.md) | `h2c-converter-*` |
| **Provider** | same as converter | [Writing converters](writing-converters.md) + [CRD patterns](writing-operators.md) | `h2c-provider-*` |
| **Transform** | `transform()`, no `kinds` | [Writing transforms](writing-transforms.md) | `h2c-transform-*` |
| **Ingress rewriter** | `name` + `match()` + `rewrite()` | [Writing rewriters](writing-rewriters.md) | `h2c-rewriter-*` |

Converters and providers share the same code interface. The naming convention signals what they produce: converters produce synthetic resources (Secrets, ConfigMaps), providers produce compose services. See the [Extension catalogue](../../catalogue.md) for the full list.

## Available imports

```python
from helmfile2compose import ConvertContext          # passed to convert() / rewrite()
from helmfile2compose import ConvertResult           # return type
from helmfile2compose import IngressRewriter         # base class for ingress rewriters
from helmfile2compose import get_ingress_class       # resolve ingressClassName + ingressTypes
from helmfile2compose import resolve_backend         # v1/v1beta1 backend → upstream dict
from helmfile2compose import apply_replacements      # apply user-defined string replacements
from helmfile2compose import resolve_env             # resolve env/envFrom into flat list
```

These are the **pacts** — the [sacred contracts](../core-architecture.md#pacts--the-sacred-contracts) — and are stable across minor versions. Both import paths work:

```python
from helmfile2compose import ConvertContext           # via re-export
from helmfile2compose.pacts import ConvertContext     # explicit
```

- **`ConvertResult`** — return type for `convert()`. Two fields: `services` (dict) and `caddy_entries` (list).
- **`IngressRewriter`** — base class for ingress rewriters. Subclass it or implement the same duck-typed contract.
- **`get_ingress_class(manifest, ingress_types)`** — resolves `ingressClassName` from spec or annotation, then through the `ingressTypes` config mapping.
- **`resolve_backend(path_entry, manifest, ctx)`** — resolves a v1/v1beta1 Ingress backend to `{svc_name, compose_name, container_port, upstream, ns}`.
- **`apply_replacements(text, replacements)`** — applies user-defined `replacements` (from `ctx.replacements`) to a string.
- **`resolve_env(container, configmaps, secrets, workload_name, warnings, replacements=None, service_port_map=None)`** — resolves a container's `env` and `envFrom` into a flat `list[dict]` of `{name, value}` pairs.

The `_`-prefixed functions (`_apply_port_remap`, `_apply_alias_map`, etc.) still exist but may change between versions. Pin your h2c-core version if you depend on them. Transforms in particular should avoid importing from the core — see [Writing transforms](writing-transforms.md#self-contained--no-core-imports).

## Testing locally

All extension types are loaded from the same `--extensions-dir`. The loader detects each type automatically — converters, transforms, and rewriters can coexist in the same directory.

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

For distribution via [h2c-manager](../../maintainer/h2c-manager.md), each extension is a GitHub repo with:

```
h2c-{type}-{name}/
├── {name}.py              # extension class (mandatory)
├── requirements.txt       # Python deps, if any (optional)
└── README.md              # description, kinds/purpose, usage (mandatory)
```

The `.py` file must be in the repo root. `requirements.txt` follows pip format — h2c-manager checks if deps are installed and warns if not.

The README should cover: what the extension does, handled kinds (for converters/providers), dependencies, priority, usage example.

## Publishing

1. Create a GitHub repo under the `helmfile2compose` org (or your own account).
2. Create a GitHub Release with a tag (e.g. `v0.1.0`). The release doesn't need assets — the tag is what matters.
3. Open a PR to `helmfile2compose/h2c-manager` adding your extension to `extensions.json`:

```json
{
  "schema_version": 1,
  "extensions": {
    "my-extension": {
      "repo": "helmfile2compose/h2c-{type}-{name}",
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
python3 h2c-manager.py my-extension
```
