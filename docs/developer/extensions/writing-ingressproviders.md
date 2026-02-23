# Writing ingress providers

An ingress provider is the abstract base for reverse proxy backends. Subclasses implement `build_service()` and `write_config()` to produce a reverse proxy compose service and its configuration file.

> *The gatekeeper was not merely a guard — he was the gate itself, the threshold, and the architect of the path beyond. When the temple replaced him, it did not hire a new guard. It built a new door.*
>
> — *Unaussprechlichen Kulten, On the Replacement of Thresholds (if memory serves)*

## What it is

`IngressProvider` is an abstract class (subclass of `Provider`) that handles the entire Ingress conversion lifecycle. It lives in `h2c.core.ingress` internally, but extensions should import it from the pacts API:

1. Receives all `Ingress` manifests (via `convert()`)
2. Dispatches each manifest to the first matching `IngressRewriter` (via `_find_rewriter()`)
3. Collects all rewriter entries
4. Calls `build_service(entries, ctx)` to produce the reverse proxy compose service
5. Calls `write_config(entries, output_dir, config)` to write the proxy config file

```python
from h2c import IngressProvider
```

## Base class attributes

| Attribute | Value | Description |
|-----------|-------|-------------|
| `kinds` | `["Ingress"]` | Hardcoded — IngressProvider always claims Ingress |
| `priority` | `900` | Runs after all other converters/providers |
| `name` | `"ingress"` | Override with your provider's name (e.g. `"caddy"`, `"traefik"`) |

## The contract

Two methods to implement:

### `build_service(entries, ctx) -> dict`

Receives all collected rewriter entries and the `ConvertContext`. Returns a dict of compose service definitions (keyed by service name), or empty dict if disabled.

```python
def build_service(self, entries, ctx):
    return {"my-proxy": {
        "image": "my-proxy:latest",
        "restart": "always",
        "ports": ["80:80", "443:443"],
        "volumes": ["./my-proxy.conf:/etc/proxy.conf:ro"],
    }}
```

### `write_config(entries, output_dir, config)`

Writes the reverse proxy configuration file to `output_dir`. Receives the collected entries, the output directory, and the raw config dict (from `helmfile2compose.yaml`).

```python
def write_config(self, entries, output_dir, config):
    import os
    path = os.path.join(output_dir, "my-proxy.conf")
    with open(path, "w", encoding="utf-8") as f:
        for entry in entries:
            f.write(f"route {entry['host']}{entry['path']} -> {entry['upstream']}\n")
```

## How dispatch works

The `convert()` method on `IngressProvider` is already implemented — you don't override it. It does:

```python
def convert(self, _kind, manifests, ctx):
    entries = []
    for m in manifests:
        rewriter = self._find_rewriter(m, ctx)
        entries.extend(rewriter.rewrite(m, ctx))
    services = {}
    if entries and not ctx.config.get("disable_ingress"):
        services = self.build_service(entries, ctx)
    return ProviderResult(services=services, ingress_entries=entries)
```

Each Ingress manifest is matched against loaded `IngressRewriter` classes. The rewriter translates controller-specific annotations into a list of entry dicts (see [Writing rewriters](writing-rewriters.md) for the entry format). Your provider then consumes those entries to build the service and config.

## Reference: CaddyProvider

The built-in `CaddyProvider` (in the helmfile2compose distribution's `extensions/caddy.py`) is the reference implementation:

```python
from h2c import IngressProvider

class CaddyProvider(IngressProvider):
    name = "caddy"

    def build_service(self, entries, ctx):
        ext_cfg = ctx.extension_config
        if ext_cfg.get("disabled"):
            return {}
        volume_root = ctx.config.get("volume_root", "./data")
        caddy_volumes = [
            "./Caddyfile:/etc/caddy/Caddyfile:ro",
            f"{volume_root}/caddy:/data",
            f"{volume_root}/caddy-config:/config",
        ]
        # Mount CA secrets for backend TLS
        ca_secrets = {e["server_ca_secret"] for e in entries
                      if e.get("server_ca_secret")}
        for secret_name in sorted(ca_secrets):
            caddy_volumes.append(
                f"./secrets/{secret_name}"
                f":/etc/caddy/certs/{secret_name}:ro")
        return {"caddy": {
            "image": "caddy:2-alpine", "restart": "always",
            "ports": ["80:80", "443:443"],
            "volumes": caddy_volumes,
        }}

    def write_config(self, entries, output_dir, config):
        # Writes a Caddyfile with host blocks, TLS config,
        # reverse_proxy directives, strip_prefix, extra_directives
        ...
```

## When to write one

Write an `IngressProvider` when you want to replace the entire reverse proxy backend — Caddy with Traefik, Nginx Proxy Manager, HAProxy (as a proxy, not just a rewriter), etc.

If you only need to translate a different ingress controller's annotations into entry dicts, write an [IngressRewriter](writing-rewriters.md) instead — that's much simpler.

## Distribution-level vs external

`IngressProvider` subclasses are *typically* bundled into distributions at build time via `build-distribution.py` — that's the intended pattern, since the ingress provider is a core architectural choice for a distribution.

That said, loading an `IngressProvider` from `--extensions-dir` works — the extension loader detects it as a converter (it has `kinds` and `convert()`), and the CLI scans for the active `IngressProvider` at runtime. It's just not the recommended path: the ingress provider shapes the entire output (Caddyfile vs nginx.conf vs traefik.yml), so shipping it as a loose external extension makes the setup fragile. Prefer building a [custom distribution](../distributions.md) instead.

```
my-distribution/
├── extensions/
│   ├── caddy.py             # or traefik_proxy.py, nginx_pm.py, etc.
│   ├── workloads.py
│   └── ...
├── .github/workflows/
│   └── release.yml
└── README.md
```
