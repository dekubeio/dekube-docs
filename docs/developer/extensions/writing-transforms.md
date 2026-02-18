# Writing transforms

A transform is an extension that post-processes the final compose output — after converters have produced services, after network aliases have been injected, after overrides and hostname truncation. Transforms see the finished result and reshape it.

Converters answer the question "what does this K8s manifest become in compose?" Transforms answer a different one: "now that everything is assembled, what needs to change?"

> *The builders laid every stone according to the plans. Then the mason arrived — not to add stones, but to chisel the ones already placed. The builders protested: the temple was complete. The mason replied: "Complete, yes. Inhabitable, no."*
>
> — *Voynich Manuscript, On the Mason's Privilege (or so the rites suggest)*

## The contract

A transform class must have:

1. **`transform(compose_services, caddy_entries, ctx)`** — called once after all converters, mutates in place
2. **No `kinds` attribute** — this is what the loader uses to distinguish transforms from converters

```python
class MyTransform:
    priority = 100  # optional, default 100, lower = earlier

    def transform(self, compose_services, caddy_entries, ctx):
        for svc_name, svc in compose_services.items():
            env = svc.get("environment", {})
            # ... post-processing logic
```

That's it. No return value — transforms mutate `compose_services` and `caddy_entries` in place. This time, in a controlled manner. In theory. Hopefully.

### Arguments

| Argument | Type | Description |
|----------|------|-------------|
| `compose_services` | `dict[str, dict]` | All compose service definitions. Keyed by service name. Mutable. |
| `caddy_entries` | `list[dict]` | Caddy reverse proxy entries. Each has `host`, `path`, `upstream`, `scheme`, optionally `server_ca_secret`, `server_sni`, `strip_prefix`, `extra_directives`. Mutable. |
| `ctx` | `ConvertContext` | Same context as converters. See [ConvertContext](writing-converters.md#convertcontext-ctx) for all attributes. |

### When transforms run

Transforms execute at the end of `convert()`, after:

1. All converters have produced services
2. Network aliases have been injected (`networks.default.aliases`)
3. `_generate_fix_permissions` has run
4. Overrides from `helmfile2compose.yaml` have been applied
5. Hostname truncation (>63 chars) has been applied

This means transforms see the *final* compose output — aliases, overrides, fix-permissions services, everything. They are the last step before the output is written to disk.

### Priority

Same system as converters. Set `priority` as a class attribute. Lower = earlier. Default: 100.

```python
class EarlyTransform:
    priority = 50   # runs before default transforms

class LateTransform:
    priority = 200  # runs after default transforms
```

Priority matters when multiple transforms are loaded and one depends on another's mutations.

## What transforms can do

Transforms have full access to `compose_services`, `caddy_entries`, and `ctx`. They can:

- **Add, remove, or modify services** — inject a monitoring sidecar, strip debug services, rewrite images
- **Rewrite environment variables** — search-and-replace across all services
- **Modify Caddy entries** — rewrite upstreams, add headers, change routing
- **Modify files on disk** — `ctx.output_dir` points to the output directory where configmaps and secrets have already been written
- **Read context** — `ctx.alias_map`, `ctx.services_by_selector`, `ctx.config` are all available for decision-making

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
    priority = 200

    def transform(self, compose_services, caddy_entries, ctx):
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

        # Rewrite Caddy upstreams
        for entry in caddy_entries:
            entry["upstream"] = _rewrite_text(
                entry["upstream"], ctx.alias_map)
```

Note the three rewrite targets: environment variables, configmap files on disk (`_rewrite_configmap_files`), and Caddy upstreams. The module-level functions (`_rewrite_text`, `_rewrite_k8s_dns`, `_apply_alias_map`) are self-contained — see [Self-contained — no core imports](#self-contained--no-core-imports).

### Example: inject a service

```python
class AddWhoami:
    """Add a whoami debug service to every compose output."""

    def transform(self, compose_services, caddy_entries, ctx):
        compose_services["whoami"] = {
            "image": "traefik/whoami:latest",
            "restart": "always",
        }
```

## Self-contained — no core imports

Transforms should not import private functions from `helmfile2compose`. The core's `_`-prefixed functions (`_apply_port_remap`, `_K8S_DNS_RE`, etc.) are internal and may change between versions.

If your transform needs regex patterns or utility functions that exist in the core, copy them into your module. This is intentional — transforms are independent modules that should work across core versions without breakage. `flatten-internal-urls` defines its own `_K8S_DNS_RE` and `_apply_alias_map` for this reason.

The public API (`ConvertResult`, `apply_replacements`, `resolve_env`) is available for import but rarely useful in transforms — those are converter-oriented tools.

See [Writing extensions](index.md) for testing, repo structure, publishing, and available imports.
