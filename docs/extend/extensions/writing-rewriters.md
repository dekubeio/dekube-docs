# Writing rewriters

Every ingress controller invented its own annotation language. HAProxy puts path rewrites in `haproxy.org/path-rewrite`. Nginx puts them in `nginx.ingress.kubernetes.io/rewrite-target`. Traefik decided annotations were beneath it and uses CRDs instead (but also supports annotations, because consistency is for the weak). A rewriter translates one controller's dialect into provider-agnostic ingress entries that any `IngressProvider` can consume.

> *The gatekeepers of old each bore a different sigil, yet all opened the same threshold. The pilgrim need not know the sigil — only declare which gate he approaches, and the keeper shall answer in kind.*
>
> — *Cultes des Goules, On the Many Gates (on good authority)*

The contract is small — three methods, one name — because the hard part isn't the interface. The hard part is reading the annotations of an ingress controller you didn't choose, translating semantics that were never designed to be portable, and producing something that a completely different reverse proxy can understand. The contract just gives you a place to put that suffering.

## The contract

A rewriter class must have:

1. **`name`** — a string identifying the rewriter (e.g. `"haproxy"`, `"nginx"`). Used for override matching: an external rewriter with the same `name` as a built-in one replaces it.
2. **`match(manifest, ctx)`** — return `True` if this rewriter handles this Ingress manifest. Typically checks `ingressClassName` (resolved through `ingress_types` config) or annotation prefixes.
3. **`rewrite(manifest, ctx)`** — convert one Ingress manifest to a list of entry dicts (see [entry format](#entry-format) below).
4. **`priority`** *(optional)* — integer, default `1000`. Lower = checked earlier. External rewriters are always checked before built-in ones, regardless of priority. Priority only orders rewriters within the same pool (external or built-in).

```python
from dekube import IngressRewriter, get_ingress_class, resolve_backend

class NginxRewriter(IngressRewriter):
    name = "nginx"

    def match(self, manifest, ctx):
        ingress_types = ctx.config.get("ingress_types") or {}
        cls = get_ingress_class(manifest, ingress_types)
        if cls == "nginx":
            return True
        annotations = manifest.get("metadata", {}).get("annotations", {})
        return any(k.startswith("nginx.ingress.kubernetes.io/") for k in annotations)

    def rewrite(self, manifest, ctx):
        entries = []
        for rule in (manifest.get("spec") or {}).get("rules") or []:
            host = rule.get("host", "")
            if not host:
                continue
            for path_entry in (rule.get("http") or {}).get("paths") or []:
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
| `strip_prefix` | `str` | no | Path prefix to strip before proxying. On multi-path rules, scope it to the matching path — don't apply a global annotation blindly to every entry. |
| `response_headers` | `dict[str, str]` | no | Headers to add to responses (e.g. CORS headers, security headers) |
| `max_body_size` | `str` | no | Max client body size (e.g. `"100M"`) |
| `extra_directives` | `list[str]` | no | **Deprecated.** Provider-specific raw directives. See below. |

### Structured fields vs `extra_directives`

Prefer `response_headers` and `max_body_size` over `extra_directives`. Structured fields are provider-agnostic — every `IngressProvider` (Caddy, Nginx, future providers) can consume them. `extra_directives` was Caddy-specific syntax that other providers couldn't interpret.

```python
entries.append({
    "host": "app.example.com",
    "path": "/",
    "upstream": "app:8080",
    "scheme": "http",
    "response_headers": {
        "X-Frame-Options": "DENY",
        "Access-Control-Allow-Origin": "*",
    },
    "max_body_size": "100M",
})
```

`extra_directives` still works — the CaddyProvider reads it as a fallback for third-party rewriters that haven't migrated. New rewriters should use structured fields exclusively.

## How dispatch works

When the `IngressProvider` processes Ingress manifests, each manifest is dispatched to the first matching rewriter:

1. External rewriters are checked first (in priority order)
2. Built-in rewriters are checked next
3. If no rewriter matches, a warning is emitted and the manifest is skipped

The built-in `HAProxyRewriter` matches:

- `ingressClassName: haproxy` or empty/absent class (acts as default fallback)
- Any manifest with `haproxy.org/*` annotations

### Custom ingress class names (`ingress_types`)

When clusters use custom `ingressClassName` values (e.g. `haproxy-controller-internal`, `nginx-external`), add an `ingress_types` mapping in `dekube.yaml` to resolve them to canonical rewriter names:

```yaml
ingress_types:
  haproxy-controller-internal: haproxy
  haproxy-controller-external: haproxy
  nginx-internal: nginx
```

The mapping is applied before rewriter dispatch — rewriters see the canonical name. Without it, custom class names won't match any rewriter and the Ingress is skipped with a warning.

Inside your rewriter, use `get_ingress_class(manifest, ctx.config.get("ingress_types") or {})` to get the resolved class name. Both `get_ingress_class` and `resolve_backend` are part of the public interface — import them from `dekube`.

For building a complete reverse proxy backend (not just an annotation translator), see [Writing ingress providers](writing-ingressproviders.md).

## Override mechanism

An external rewriter with the same `name` as a built-in one replaces it entirely. For example, a custom `HAProxyRewriter` with `name = "haproxy"` would replace the built-in HAProxy handling.

When an override occurs, dekube prints:

```
Rewriter overrides built-in: haproxy
```

See [Writing extensions](index.md) for testing, repo structure, publishing, and available imports.
