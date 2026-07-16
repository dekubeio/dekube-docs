---
description: "How to write a dekube extension: choosing between converter, provider, transform, rewriter, or ingress provider, with available imports."
---

# Writing extensions

So you want to teach dekube a new heresy. Take a moment to reconsider. Then, having failed to reconsider, make sure you're familiar with [Concepts](../../understand/concepts.md) (design philosophy, emulation boundary) and [Architecture](../../understand/architecture.md) (converter pipeline, dispatch loop). A look at [Code quality](../code-quality.md) is also recommended — the bar is higher than the project's origins would suggest.

> *The acolyte approached the altar and asked: "May I add my own prayer to the liturgy?" The high priest did not refuse. The high priest never refuses. That is the problem.*
>
> — *Book of Eibon, On Open Extension Points (broadly speaking)*

!!! tip "Something broken?"
    If the engine contract doesn't behave as documented, or if you hit a bug while developing an extension, open an issue on [dekube-engine](https://github.com/dekubeio/dekube-engine/issues). If the problem is in a bundled extension (one of the Eight Monks), file it on the extension's own repo — each one is linked in the [catalogue](../../catalogue.md). Not sure where the bug lives? Use [helmfile2compose](https://github.com/dekubeio/helmfile2compose/issues) — I'll triage.

## Which extension type do I need?

- **My K8s manifests contain a CRD that dekube doesn't know about** → write a [**Converter**](writing-converters.md) (resource-only) or a [**Provider**](writing-providers.md) (if it should produce compose services)
- **I need to populate `ctx` lookups from K8s manifests without producing compose services** → write an [**Indexer**](writing-converters.md#indexers) (subclass of `IndexerConverter`)
- **The compose output is correct but I need to post-process it** (rewrite env vars, inject services, fix permissions) → write a [**Transform**](writing-transforms.md)
- **My cluster uses an ingress controller whose annotations aren't supported** → write an [**Ingress rewriter**](writing-rewriters.md)
- **I want to replace Caddy with a different reverse proxy** → write an [**Ingress provider**](writing-ingressproviders.md)

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
from dekube import secret_value             # decode a Secret key (base64 or plain)

# Extension helper functions (stable API)
from dekube import log                       # print "  [name] msg" to stderr
from dekube import generate_password         # random alphanumeric password
from dekube import is_excluded               # fnmatch a name against exclude patterns
from dekube import iter_workloads            # yield (name, pod_spec) for every workload
from dekube import iter_named_containers     # yield (compose_name, container): main/init/sidecar
from dekube import apply_alias_map           # rewrite K8s Service names → compose names in hostnames
from dekube import rewrite_k8s_dns           # <svc>.<ns>.svc[.cluster.local] → <svc>
from dekube import write_configmap_files     # emit a ConfigMap's data to configmaps/<name>/
from dekube import write_secret_files        # emit a Secret's data to secrets/<name>/

# K8s-to-compose conversion primitives (stable API)
from dekube import convert_command          # K8s command/args → compose entrypoint/command
from dekube import convert_volume_mounts    # volumeMounts → compose volume strings
from dekube import build_alias_map          # K8s Service names → compose service names
from dekube import build_service_port_map   # (service, port) → container port
from dekube import resolve_named_port       # named port → numeric containerPort
```

These are all stable across minor versions. Everything above the `# K8s-to-compose conversion primitives` comment — base classes, result types, and helper functions — are **pacts** (they live in `dekube.pacts`), the [sacred contracts](../../understand/engine.md#pacts--the-sacred-contracts). Both import paths work for them:

```python
from dekube import ConvertContext           # via re-export
from dekube.pacts import ConvertContext     # explicit
```

The conversion primitives (`convert_command`, `convert_volume_mounts`, `build_alias_map`, `build_service_port_map`, `resolve_named_port`) and `IngressProvider` live in `dekube.core` — import them from `dekube` only.

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
- **`secret_value(secret, key)`** — decodes a single key from a K8s Secret dict. Handles both `stringData` (plain text) and `data` (base64-decoded). Returns `str | None`. Useful for converters that need to read Secret values injected by other converters (e.g. reading a database password from a cert-manager-generated secret).
- **`log(name, msg)`** — prints `  [name] msg` to stderr. Use `log("my-extension", "...")` instead of defining your own `_log` — a free function can't collide with another extension's (see [Keep helpers inside the class](#keep-helpers-inside-the-class)).
- **`generate_password(length=24)`** — returns a random alphanumeric password. Providers emulating an operator that auto-generates credentials (cnpg, keycloak) use this. Pass an explicit `length` if the operator's default differs.
- **`is_excluded(name, patterns)`** — `True` if `name` matches any `fnmatch` pattern in `patterns` (null-safe: `None` patterns → `False`). The exclusion-glob every workload-producing extension needs.
- **`iter_workloads(manifests)`** — yields `(workload_name, pod_spec)` for every workload manifest (Deployment, StatefulSet, DaemonSet, Job, Pod, …). Null-safe against Helm's `null`-rendered lists. `manifests` is `ctx.manifests`.
- **`iter_named_containers(name, pod_spec)`** — yields `(compose_service_name, container)` for a pod's main, init, and sidecar containers, named the way the workload converter names them (`name`, `name-init-<c>`, `name-sidecar-<c>`). Pair with `iter_workloads` to walk every container in the output.
- **`apply_alias_map(text, alias_map)`** — rewrites K8s Service names to compose service names in hostname positions (preceded by `/` or `@`, followed by `/ : whitespace`, quotes, or end). Only hostnames are touched, not substrings like bucket names. Pass `ctx.alias_map`.
- **`rewrite_k8s_dns(text)`** — collapses `<svc>.<ns>.svc[.cluster.local][:port]` down to `<svc>`. Used by the flatten-internal-urls transform.
- **`write_configmap_files(name, ctx, items=None)`** — emits ConfigMap `name`'s data (and `binaryData`) as files under `output_dir/configmaps/<name>/`, records it in `ctx.generated_cms`, and returns the relative dir (`./configmaps/<name>`) — or `None` (appending to `ctx.warnings`) if the ConfigMap isn't in `ctx.configmaps`. Replaces hand-rolling `os.makedirs` + `open` (see [Injecting synthetic resources](writing-providers.md#injecting-synthetic-resources)).
- **`write_secret_files(name, ctx, items=None)`** — the Secret counterpart: emits `ctx.secrets[name]`'s data under `output_dir/secrets/<name>/`, records it in `ctx.generated_secrets`, returns `./secrets/<name>` or `None`.
- **`convert_command(container, env_dict)`** — converts K8s `command`/`args` to compose `entrypoint`/`command` with `$(VAR)` resolution and `$VAR` escaping.
- **`convert_volume_mounts(volume_mounts, pod_volumes, pvc_names, config, workload_name, warnings, ...)`** — converts `volumeMounts` to compose volume strings, handling PVC, ConfigMap, Secret, and emptyDir mounts.
- **`build_alias_map(manifests, services_by_selector)`** — builds a map of K8s Service names to compose service names (ClusterIP + ExternalName resolution).
- **`build_service_port_map(manifests, services_by_selector)`** — builds a map of `(service_name, service_port)` to `container_port` for port remapping.
- **`resolve_named_port(name, container_ports)`** — resolves a named port (e.g. `'http'`) to its numeric `containerPort`.

!!! note "Deprecated `_`-prefixed aliases"
    The old `_secret_value`, `_convert_command`, `_convert_volume_mounts`, `_build_alias_map`, `_build_service_port_map`, `_resolve_named_port` names still work (exported in `__all__`) for backward compatibility. Prefer the unprefixed names in new code. The helpers promoted later (`log`, `generate_password`, `is_excluded`, `iter_workloads`, `iter_named_containers`, `apply_alias_map`, `rewrite_k8s_dns`, `write_configmap_files`, `write_secret_files`) have **no** `_`-prefixed alias — they were never public under another name, so import them exactly as spelled.

Internal functions (`_apply_port_remap`, `_resolve_env_entry`, `_build_vol_map`, etc.) are not part of the stable API and may change between versions. Pin your dekube-engine version if you depend on them. Transforms in particular should avoid importing from the core — see [Writing transforms](writing-transforms.md#self-contained--no-core-imports).

## Keep helpers inside the class

Distributions concatenate all extension `.py` files into a single script. Top-level function names share a flat namespace — if two extensions define `_log()`, the second silently overwrites the first. The build system detects this and refuses to build (unless you pass `--my-extensions-are-fine-i-swear`).

!!! tip "First, reach for the engine's helpers"
    Before writing a helper at all, check [Available imports](#available-imports) — the common ones are already there. `log("my-extension", msg)` replaces a hand-rolled `_log`; `generate_password`, `is_excluded`, `iter_workloads`, `iter_named_containers` cover the usual cases. Engine functions live in the `dekube` namespace, so they can't collide with an extension's top-level names. What you don't define can't clash.

The fix, for helpers the engine doesn't provide: put them inside your class.

```python
# Good — no collisions possible
class MyTransform:
    name = "my-transform"
    priority = 100

    def _log(self, msg):
        print(f"  [{self.name}] {msg}", file=sys.stderr)

    @staticmethod
    def _parse_thing(data):
        ...

    def transform(self, compose_services, ingress_entries, ctx):
        self._log("doing things")
        result = self._parse_thing(data)
```

- Methods that need `self` (logging with `self.name`) → regular methods
- Pure helpers → `@staticmethod`
- Call via `self._func()` from instance methods, `ClassName._func()` from static methods
- Top-level constants (`_WORKLOAD_KINDS = (...)`) are fine — only functions collide

This applies to all extension types: converters, providers, transforms, rewriters.

## Input validation: not your problem

The engine assumes its input manifests are valid Kubernetes YAML — it does [zero validation](../../understand/concepts.md#no-validation-by-design) and your extension shouldn't either. If a manifest is missing a field, has the wrong type, or references something that doesn't exist, that's a broken helmfile, not your bug. You don't need to guard against malformed input.

If you *want* to handle edge cases gracefully in your extension, that's your call — but the engine won't help you. No schema validation, no error wrapping, no safety net. A missing key is a `KeyError` and that's fine.

## Quickstart: writing a converter from scratch

From an empty file to a working extension. The ritual is short — the consequences are not.

**Scenario:** You want to handle a `RedisCluster` CRD that produces a Redis compose service.

### 1. Create the file

```
dekube-provider-redis-cluster/
├── redis_cluster.py
└── README.md
```

### 2. Write the extension

```python
# redis_cluster.py
from dekube import Provider, ProviderResult

class RedisClusterProvider(Provider):
    kinds = ["RedisCluster"]
    name = "redis-cluster"

    def convert(self, kind, manifests, ctx):
        services = {}
        for m in manifests:
            name = m.get("metadata", {}).get("name", "redis")
            spec = m.get("spec") or {}
            ns = m.get("metadata", {}).get("namespace", "default")
            services[name] = {
                "image": f"redis:{spec.get('version', '7')}-alpine",
                "restart": "always",
                "command": ["redis-server", "--requirepass", spec.get("password", "changeme")],
            }
            # Register the Service so network aliases are generated
            ctx.services_by_selector[name] = {
                "name": name, "namespace": ns,
                "selector": {}, "type": "ClusterIP",
                "ports": [{"port": 6379, "targetPort": 6379}],
            }
        return ProviderResult(services=services)
```

### 3. Test locally

Create a test manifest:

```yaml
# /tmp/test-manifests/redis.yaml
apiVersion: redis.example.com/v1
kind: RedisCluster
metadata:
  name: my-redis
  namespace: cache
spec:
  version: "7"
  password: s3cret
```

Run with the distribution:

```bash
python3 helmfile2compose.py \
  --from-dir /tmp/test-manifests \
  --extensions-dir ./dekube-provider-redis-cluster \
  --output-dir /tmp/output
```

Check the output:

```bash
cat /tmp/output/compose.yml
# Should contain a my-redis service with redis:7-alpine
```

### 4. Publish

Create a GitHub repo, tag a release, and submit a PR to [dekube-manager's `extensions.json`](https://github.com/dekubeio/dekube-manager). See [Publishing](#publishing) below.

---

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
