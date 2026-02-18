# Writing rewriters

An ingress rewriter translates controller-specific Ingress annotations into Caddy configuration entries. Each rewriter targets a specific ingress controller — dispatched by `ingressClassName` or annotation prefix.

> *The gatekeepers of old each bore a different sigil, yet all opened the same threshold. The pilgrim need not know the sigil — only declare which gate he approaches, and the keeper shall answer in kind.*
>
> — *Cultes des Goules, On the Many Gates (on good authority)*

## The contract

A rewriter class must have:

1. **`name`** — a string identifying the rewriter (e.g. `"haproxy"`, `"nginx"`). Used for override matching: an external rewriter with the same `name` as a built-in one replaces it.
2. **`match(manifest, ctx)`** — return `True` if this rewriter handles this Ingress manifest. Typically checks `ingressClassName` (resolved through `ingressTypes` config) or annotation prefixes.
3. **`rewrite(manifest, ctx)`** — convert one Ingress manifest to a list of Caddy entry dicts.
4. **`priority`** *(optional)* — integer, default `100`. Lower = checked earlier. External rewriters are always checked before built-in ones, regardless of priority. Priority only orders rewriters within the same pool (external or built-in).

```python
from helmfile2compose import IngressRewriter, get_ingress_class, resolve_backend

class NginxRewriter(IngressRewriter):
    name = "nginx"

    def match(self, manifest, ctx):
        ingress_types = ctx.config.get("ingressTypes", {})
        cls = get_ingress_class(manifest, ingress_types)
        if cls == "nginx":
            return True
        annotations = manifest.get("metadata", {}).get("annotations", {})
        return any(k.startswith("nginx.ingress.kubernetes.io/") for k in annotations)

    def rewrite(self, manifest, ctx):
        entries = []
        for rule in manifest.get("spec", {}).get("rules", []):
            host = rule.get("host", "")
            if not host:
                continue
            for path_entry in rule.get("http", {}).get("paths", []):
                backend = resolve_backend(path_entry, manifest, ctx)
                entries.append({
                    "host": host,
                    "path": path_entry.get("path", "/"),
                    "upstream": backend["upstream"],
                    "scheme": "http",
                })
        return entries
```

### Entry format

Each entry dict returned by `rewrite()` must have:

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `host` | `str` | yes | The hostname (e.g. `app.example.com`) |
| `path` | `str` | yes | The path (`/` for catch-all, `/api` for specific) |
| `upstream` | `str` | yes | The upstream address (`host:port`) |
| `scheme` | `str` | yes | `http` or `https` |
| `server_ca_secret` | `str` | no | Secret name containing CA cert for backend TLS |
| `server_sni` | `str` | no | SNI server name for backend TLS |
| `strip_prefix` | `str` | no | Path prefix to strip before proxying |
| `extra_directives` | `list[str]` | no | Raw Caddy directive strings (see below) |

### `extra_directives`

A list of raw Caddy directive strings injected into the host block. They are written after `uri strip_prefix` and before `reverse_proxy`, which places them in the right Caddy order for directives like `header`, `rate_limit`, `basic_auth`, `forward_auth`, `request_body`, etc.

```python
entries.append({
    "host": "app.example.com",
    "path": "/",
    "upstream": "app:8080",
    "scheme": "http",
    "extra_directives": [
        "header X-Frame-Options DENY",
        "rate_limit {remote.ip} 100r/m",
    ],
})
```

## How dispatch works

When `IngressConverter` processes Ingress manifests, each manifest is dispatched to the first matching rewriter:

1. External rewriters are checked first (in priority order)
2. Built-in rewriters are checked next
3. If no rewriter matches, a warning is emitted and the manifest is skipped

The built-in `HAProxyRewriter` matches:

- `ingressClassName: haproxy` or empty/absent class (acts as default fallback)
- Any manifest with `haproxy.org/*` annotations

### Custom ingress class names (`ingressTypes`)

When clusters use custom `ingressClassName` values (e.g. `haproxy-controller-internal`, `nginx-external`), add an `ingressTypes` mapping in `helmfile2compose.yaml` to resolve them to canonical rewriter names:

```yaml
ingressTypes:
  haproxy-controller-internal: haproxy
  haproxy-controller-external: haproxy
  nginx-internal: nginx
```

The mapping is applied before rewriter dispatch — rewriters see the canonical name. Without it, custom class names won't match any rewriter and the Ingress is skipped with a warning.

Inside your rewriter, use `get_ingress_class(manifest, ctx.config.get("ingressTypes", {}))` to get the resolved class name. Both `get_ingress_class` and `resolve_backend` are part of the public interface — import them from `helmfile2compose`.

## Override mechanism

An external rewriter with the same `name` as a built-in one replaces it entirely. For example, a custom `HAProxyRewriter` with `name = "haproxy"` would replace the built-in HAProxy handling.

When an override occurs, h2c prints:

```
Rewriter overrides built-in: haproxy
```

See [Writing extensions](index.md) for testing, repo structure, publishing, and available imports.
