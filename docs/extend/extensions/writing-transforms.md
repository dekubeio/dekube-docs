# Writing transforms

A transform sees the compose output after everyone else is done — converters have produced services, aliases have been injected, hostnames have been truncated — and decides what still needs to change. It can rewrite environment variables, inject services, strip aliases, fix permissions, or do anything else that doesn't fit in a converter's "one kind in, resources out" model. Overrides run *after* transforms, so even the transform's work can be overridden in `dekube.yaml`. Nothing is final until the user says so.

Converters answer the question "what does this K8s manifest become in compose?" Transforms answer a different one: "now that everything is assembled, what needs to change?"

> *The builders laid every stone according to the plans. Then the mason arrived — not to add stones, but to chisel the ones already placed. The builders protested: the temple was complete. The mason replied: "Complete, yes. Inhabitable, no."*
>
> — *Voynich Manuscript, On the Mason's Privilege (or so the rites suggest)*

## The contract

Transforms use duck typing — no base class. The extension loader detects a transform by checking for a `transform()` method **and** the absence of a `kinds` attribute (which would make it a converter). This keeps transform files minimal and independent of the engine's class hierarchy.

| Attribute | Required | Description |
|-----------|----------|-------------|
| `transform(self, compose_services, ingress_entries, ctx)` | **yes** | Called once after all converters. Mutates in place, no return value. |
| `kinds` | **must be absent** | Presence of `kinds` makes the loader treat the class as a converter. |
| `priority` | optional | `int`, default 1000. Lower = earlier. |
| `name` | optional | `str`. Used to match `extensions.<name>.enabled: false` in `dekube.yaml`. |

```python
class MyTransform:
    name = "my-transform"
    priority = 1000  # optional, default 1000, lower = earlier

    def transform(self, compose_services, ingress_entries, ctx):
        for svc_name, svc in compose_services.items():
            env = svc.get("environment") or {}
            # ... post-processing logic
```

That's it. No return value — transforms mutate `compose_services` and `ingress_entries` in place. This time, in a controlled manner. In theory. Hopefully.

### Arguments

| Argument | Type | Description |
|----------|------|-------------|
| `compose_services` | `dict[str, dict]` | All compose service definitions. Keyed by service name. Mutable. |
| `ingress_entries` | `list[dict]` | Ingress entries consumed by the configured `IngressProvider`. Each has `host`, `path`, `upstream`, `scheme`, optionally `server_ca_secret`, `server_sni`, `strip_prefix`, `response_headers`, `max_body_size`. Mutable. |
| `ctx` | `ConvertContext` | Same context as converters. See [ConvertContext](writing-converters.md#convertcontext-ctx) for all attributes. |

### When transforms run

Transforms execute near the end of `convert()`, after:

1. All converters have produced services
2. Network aliases have been injected (`networks.default.aliases`)
3. Hostname truncation (>63 chars) has been applied

And *before*:

4. Overrides from `dekube.yaml` are applied (after transforms)

This means transforms see the assembled compose output — aliases, truncated hostnames — but *not* user overrides. Overrides run last so that services created by transforms (e.g. `fix-permissions` busybox) can be overridden in `dekube.yaml`. Transforms are sorted by priority — lower runs first. The built-in `fix-permissions` transform runs at priority 8000 (after all other transforms).

### Priority

Same system as converters. Set `priority` as a class attribute. Lower = earlier. Default: 1000 (no base class for transforms — the fallback applies).

```python
class EarlyTransform:
    priority = 500   # runs before default (1000)

class LateTransform:
    priority = 2000  # runs after default (1000)
```

Priority matters when multiple transforms are loaded and one depends on another's mutations. In a system where every extension promised to be independent, it's remarkable how quickly they start depending on execution order.

## What transforms can do

Transforms have full access to `compose_services`, `ingress_entries`, and `ctx`. They can:

- **Add, remove, or modify services** — inject a monitoring sidecar, strip debug services, rewrite images
- **Rewrite environment variables** — search-and-replace across all services
- **Modify ingress entries** — rewrite upstreams, add headers, change routing
- **Modify files on disk** — `ctx.output_dir` points to the output directory where configmaps and secrets have already been written
- **Read context** — `ctx.alias_map`, `ctx.services_by_selector`, `ctx.config` are all available for decision-making

### Interaction with fix-permissions

Transforms can break each other's assumptions. Priority ordering helps, but when one transform rewrites what another inspects, the results depend on who runs first — and whether anyone anticipated the combination. The fix-permissions interaction is the canonical example.

The `fix-permissions` transform (bundled in helmfile2compose, priority 8000) generates a `chown` service for non-root containers with bind-mounted volumes. It reads the UID from the Kubernetes manifest's `securityContext.runAsUser` and compares the manifest image with the compose service image.

If your transform **changes the image** of a service, fix-permissions will detect the mismatch and skip the service — the manifest UID is no longer reliable. A warning is logged. This is the safe default: no incorrect chown is better than a wrong one.

To restore the chown with the correct UID, set `user:` on the compose service:

```python
def transform(self, compose_services, ingress_entries, ctx):
    svc = compose_services.get("my-service")
    if svc:
        svc["image"] = "custom-image:latest"
        svc["user"] = "999"  # fix-permissions will chown to this UID
```

fix-permissions resolves the effective UID in this order:

1. **`user:` on the compose service** — always wins (set by transform or user override)
2. **Manifest UID** — used only if the image hasn't changed
3. **Skip** — image changed, no `user:` set → warning, no chown

This means your transform doesn't *have to* set `user:` — fix-permissions degrades gracefully. But if you know the UID, setting it is the right thing to do.

### Example: strip network aliases

The [`flatten-internal-urls`](../../catalogue.md#flatten-internal-urls) transform is the reference implementation. Its core logic (simplified — the real implementation uses module-level functions, not methods):

```python
import re

_K8S_DNS_RE = re.compile(
    r'([a-z0-9](?:[a-z0-9-]*[a-z0-9])?)'
    r'\.[a-z0-9-]+\.svc(?:\.cluster\.local)?(?::\d+)?'
)

def _rewrite_text(text, alias_map):
    """Rewrite K8s FQDNs to short names, then apply alias map."""
    text = _K8S_DNS_RE.sub(r'\1', text)
    # ... alias_map substitution
    return text

class FlattenInternalUrls:
    priority = 2000

    def transform(self, compose_services, ingress_entries, ctx):
        # Strip aliases
        for svc in compose_services.values():
            networks = svc.get("networks")
            if isinstance(networks, dict):
                for net_cfg in networks.values():
                    if isinstance(net_cfg, dict):
                        net_cfg.pop("aliases", None)

        # Rewrite FQDNs in env vars
        for svc in compose_services.values():
            env = svc.get("environment")
            if not env or not isinstance(env, dict):
                continue
            for key in list(env):
                val = env[key]
                if isinstance(val, str):
                    env[key] = _rewrite_text(val, ctx.alias_map)

        # Rewrite configmap files on disk
        # (walks ctx.output_dir/configmaps/ and rewrites file contents)

        # Rewrite ingress upstreams
        for entry in ingress_entries:
            entry["upstream"] = _rewrite_text(
                entry["upstream"], ctx.alias_map)
```

Note the three rewrite targets: environment variables, configmap files on disk (`_rewrite_configmap_files`), and Caddy upstreams. The module-level functions (`_rewrite_text`, `_rewrite_k8s_dns`, `_apply_alias_map`) are self-contained — see [Self-contained — no core imports](#self-contained--no-core-imports).

### Example: inject a service

```python
class AddWhoami:
    """Add a whoami debug service to every compose output."""

    def transform(self, compose_services, ingress_entries, ctx):
        compose_services["whoami"] = {
            "image": "traefik/whoami:latest",
            "restart": "always",
        }
```

## Self-contained — no core imports {#self-contained--no-core-imports}

Transforms should not import private functions from `dekube`. The core's `_`-prefixed functions (`_apply_port_remap`, `_K8S_DNS_RE`, etc.) are internal and may change between versions.

If your transform needs regex patterns or utility functions that exist in the core, copy them into your module. This is intentional — transforms are independent modules that should work across core versions without breakage. `flatten-internal-urls` defines its own `_K8S_DNS_RE` and `_apply_alias_map` for this reason.

The public API (`ConvertResult`, `apply_replacements`, `resolve_env`) is available for import but rarely useful in transforms — those are converter-oriented tools.

See [Writing extensions](index.md) for testing, repo structure, publishing, and available imports.
