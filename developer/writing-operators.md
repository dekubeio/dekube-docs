# Writing operators

An operator is an external CRD converter — a Python module that teaches helmfile2compose how to handle Custom Resource Definitions. Each operator is a single `.py` file with a converter class.

> *To name a thing is to summon it. To teach another its name is to bind your fate to what answers.*
>
> — *Book of Eibon, Des Noms et des Liens (probablement)*

## The contract

A converter class must have:

1. **`kinds`** — a list of K8s kinds to handle (e.g. `["Keycloak", "KeycloakRealmImport"]`)
2. **`convert(kind, manifests, ctx)`** — called once per kind, returns a `ConvertResult`

```python
from helmfile2compose import ConvertResult

class MyConverter:
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
        return ConvertResult(services=services)
```

### `ConvertResult`

A dataclass with two fields:

- **`services`** — `dict[str, dict]`: compose service definitions to add. Keyed by service name.
- **`caddy_entries`** — `list[dict]`: Caddy reverse proxy entries (same format as IngressConverter produces).

Both default to empty. Most operators only produce `services`.

### `ConvertContext` (`ctx`)

The conversion context passed to every converter. Key attributes:

| Attribute | Type | Description |
|-----------|------|-------------|
| `ctx.configmaps` | `dict` | Indexed ConfigMaps (name -> manifest) |
| `ctx.secrets` | `dict` | Indexed Secrets (name -> manifest). **Writable** — operators can inject synthetic secrets. |
| `ctx.config` | `dict` | The `helmfile2compose.yaml` config |
| `ctx.output_dir` | `str` | Output directory for generated files |
| `ctx.warnings` | `list[str]` | Append warnings here (printed to stderr) |
| `ctx.generated_cms` | `set[str]` | Names of ConfigMaps already written to disk |
| `ctx.generated_secrets` | `set[str]` | Names of Secrets already written to disk |
| `ctx.replacements` | `list[dict]` | User-defined string replacements |
| `ctx.alias_map` | `dict` | Service alias map (K8s Service name -> workload name) |
| `ctx.service_port_map` | `dict` | Service port map ((svc_name, port) -> container_port) |

### Priority

Set `priority` as a class attribute to control execution order. Lower = earlier. Default: 100.

```python
class CertManagerConverter:
    kinds = ["Certificate", "ClusterIssuer", "Issuer"]
    priority = 10  # runs first
```

This matters when operators depend on each other's output (e.g. trust-manager needs cert-manager's generated secrets).

### Multi-kind dispatch

If your converter handles multiple kinds and needs to process them in order, use the `kind` argument:

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
            return ConvertResult()
        # kind == "MainKind" — use indexed data
        return self._process_main(manifests, ctx)
```

## Injecting synthetic resources

Operators can inject synthetic Secrets or ConfigMaps into `ctx` for other converters or the existing volume-mount machinery to consume:

```python
# Inject a synthetic Secret (cert-manager pattern)
ctx.secrets["my-tls-secret"] = {
    "metadata": {"name": "my-tls-secret"},
    "stringData": {
        "tls.crt": pem_cert,
        "tls.key": pem_key,
    },
}

# Write files to disk so volume mounts can find them
import os
secret_dir = os.path.join(ctx.output_dir, "secrets", "my-tls-secret")
os.makedirs(secret_dir, exist_ok=True)
with open(os.path.join(secret_dir, "tls.crt"), "w", encoding="utf-8") as f:
    f.write(pem_cert)
ctx.generated_secrets.add("my-tls-secret")
```

Use `stringData` (not `data`) to avoid double-encoding — the main pipeline handles base64 decoding for `data` entries, but synthetic secrets should use plain text.

## Testing locally

1. Put your `.py` file in a directory:

```
my-operators/
└── my_converter.py
```

2. Run with `--extensions-dir`:

```bash
python3 helmfile2compose.py --from-dir /tmp/rendered \
  --extensions-dir ./my-operators --output-dir ./output
```

3. Check that your kinds are loaded:

```
Loaded operators: MyConverter (MyCustomResource)
```

4. Verify the compose output includes your services.

## Repo structure

For distribution via [h2c-manager](../maintainer/h2c-manager.md):

```
h2c-operator-myresource/
├── myresource.py          # converter class (mandatory)
├── requirements.txt       # Python deps, if any (optional)
└── README.md              # description, kinds, usage (mandatory)
```

- The `.py` file must be in the repo root.
- `requirements.txt` follows pip format. h2c-manager checks if deps are installed and warns if not.
- The README should cover: handled kinds, what gets converted, dependencies, priority, usage example.

## Publishing

1. Create a GitHub repo under the `helmfile2compose` org (or your own account).
2. Create a GitHub Release with a tag (e.g. `v0.1.0`). The release doesn't need assets — the tag is what matters.
3. Open a PR to `helmfile2compose/h2c-manager` adding your operator to `extensions.json`:

```json
{
  "myresource": {
    "repo": "helmfile2compose/h2c-operator-myresource",
    "description": "MyCustomResource CRD",
    "file": "myresource.py",
    "depends": []
  }
}
```

Once merged, users can install with:

```bash
python3 h2c-manager.py myresource
```

## Available imports

Your operator can import from `helmfile2compose`:

```python
from helmfile2compose import ConvertResult           # return type
from helmfile2compose import rewrite_k8s_dns          # rewrite *.svc.cluster.local
from helmfile2compose import _apply_replacements      # apply user replacements
from helmfile2compose import _apply_port_remap        # rewrite port in URL string
from helmfile2compose import _apply_alias_map         # rewrite service aliases
```

Only `ConvertResult` is part of the stable interface. The `_`-prefixed functions work but may change between versions. Pin your h2c-core version if you depend on them.
