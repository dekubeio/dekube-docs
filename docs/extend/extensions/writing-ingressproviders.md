# Writing ingress providers

In Kubernetes, the ingress controller is a load-balanced, health-checked, horizontally scaled reverse proxy backed by an entire control plane. We replaced that steel bridge with an assembly of twigs — a single-instance reverse proxy on a single host, configured by a generated flat file. Don't expect a freight train to cross it. But you're here, reading this page, which means you've already accepted the premise. So let's build the best twigs we can.

> *The gatekeeper was not merely a guard — he was the gate itself, the threshold, and the architect of the path beyond. When the temple replaced him, it did not hire a new guard. It built a new door.*
>
> — *Unaussprechlichen Kulten, On the Replacement of Thresholds (if memory serves)*

## What it is

`IngressProvider` is an abstract class (subclass of `Provider`) that owns the entire Ingress conversion lifecycle — from raw manifests to a running reverse proxy with a config file. Everything flows through it. Every route, every TLS termination, every path rewrite ultimately becomes whatever your provider decides to emit. It lives in `dekube.core.ingress` internally, but extensions should import it from the pacts API.

The lifecycle:

1. Receives all `Ingress` manifests (via `convert()`)
2. Dispatches each manifest to the first matching `IngressRewriter` (via `_find_rewriter()`)
3. Collects all rewriter entries — the abstract routing rules, stripped of their controller-specific dialects
4. Calls `build_service(entries, ctx)` to produce the reverse proxy compose service
5. Calls `write_config(entries, output_dir, config)` to write the proxy config file

```python
from dekube import IngressProvider
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

Writes the reverse proxy configuration file to `output_dir`. Receives the collected entries, the output directory, and the raw config dict (from `dekube.yaml`).

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

## Entry format

Each entry dict produced by rewriters has these fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `host` | `str` | yes | Hostname (e.g. `api.example.com`) |
| `path` | `str` | yes | URL path (e.g. `/api/`, `/`) |
| `upstream` | `str` | yes | Backend address (`host:port`) |
| `scheme` | `str` | yes | `http` or `https` (backend protocol) |
| `strip_prefix` | `str \| None` | no | Path prefix to strip before proxying |
| `server_ca_secret` | `str` | no | Secret name containing CA cert for backend TLS |
| `server_sni` | `str` | no | SNI hostname for backend TLS |
| `response_headers` | `dict[str, str]` | no | Headers to add to responses |
| `max_body_size` | `str` | no | Max client body size (e.g. `"100M"`) |
| `extra_directives` | `list[str]` | no | **Deprecated.** Provider-specific raw directives (Caddy syntax). Kept for third-party rewriter backward compat. |

Providers should read `response_headers` and `max_body_size` first, then fall back to `extra_directives` for backward compat with third-party rewriters that haven't migrated yet.

## Reference: CaddyProvider

The built-in `CaddyProvider` (in [dekube-provider-caddy](https://github.com/dekubeio/dekube-provider-caddy)) is the reference implementation:

```python
from dekube import IngressProvider

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
        ext_cfg = config.get("extensions", {}).get(self.name, {})
        caddy_email = ext_cfg.get("email")
        tls_internal = bool(ext_cfg.get("tls_internal"))

        filename = "Caddyfile"
        if config.get("disable_ingress"):
            filename = f"Caddyfile-{config.get('name', 'project')}"

        path = os.path.join(output_dir, filename)
        with open(path, "w", encoding="utf-8") as f:
            f.write("# Generated by helmfile2compose\n\n")
            if caddy_email:
                f.write(f"{{\n\temail {caddy_email}\n}}\n\n")
            # Group entries by host, write Caddy host blocks
            # (specific paths first, catch-all last)
            by_host = {}
            for e in entries:
                by_host.setdefault(e["host"], []).append(e)
            for host, host_entries in by_host.items():
                _write_caddy_host_block(f, host, host_entries, tls_internal)
```

See the full implementation in [dekube-provider-caddy](https://github.com/dekubeio/dekube-provider-caddy/blob/main/caddy.py) (125 lines, including the host block writer and reverse_proxy directive helpers).

## When to write one

Write an `IngressProvider` when you want to replace the entire reverse proxy backend — Caddy with Traefik, Nginx Proxy Manager, HAProxy (as a proxy, not just a rewriter), etc. You're rebuilding the front door, not just changing the lock. Make sure you actually need a new door before committing to the carpentry.

If you only need to translate a different ingress controller's annotations into entry dicts, write an [IngressRewriter](writing-rewriters.md) instead — that's much simpler, and much harder to break.

## Distribution-level vs external

`IngressProvider` subclasses are *typically* bundled into distributions at build time via `build-distribution.py` — that's the intended pattern. The ingress provider isn't just another extension. It defines the shape of the output: Caddyfile or nginx.conf or traefik.yml. The gate defines the temple.

That said, loading an `IngressProvider` from `--extensions-dir` works — the extension loader detects it as a converter (it has `kinds` and `convert()`), and the CLI scans for the active `IngressProvider` at runtime. It's just not the recommended path: shipping your gateway as a loose extension means every user must remember to install it, and forgetting turns every Ingress manifest into a warning and a shrug. Prefer building a [custom distribution](../../understand/distributions.md) instead.

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
