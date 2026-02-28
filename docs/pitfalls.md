# Pitfalls

Technical gotchas for extension developers and distribution builders. If you're a user looking for help with a broken stack, see the [helmfile2compose troubleshooting guide](https://helmfile2compose.dekube.io/docs/troubleshooting/).

## Null-safe YAML access

Python's `dict.get("key", default)` returns `default` only when the key is **absent**. When the key exists with an explicit `None` value — common in Helm charts with conditional `{{ if }}` blocks — `.get()` returns `None`, not the default.

```python
# WRONG — returns None when key exists with null value
annotations = manifest.get("metadata", {}).get("annotations", {})

# RIGHT — coalesces None to empty dict
annotations = manifest.get("metadata", {}).get("annotations") or {}
```

This applies to any field that Helm may render as `null`: `annotations`, `ports`, `initContainers`, `securityContext`, `data`, `stringData`, `rules`, `selector`. Use `or {}` / `or []` for any `.get()` on a YAML field that could be explicitly null.

See [Code quality — Code gotchas](extend/code-quality.md#code-gotchas) for the full history (v2.3.1 fixed 30+ instances).

## `sys.modules` and module identity

When a distribution runs as `helmfile2compose.py`, Python sees the module as `__main__`. Extensions that `from dekube import ConverterResult` would create a separate module namespace — `dekube.ConverterResult` ≠ `__main__.ConverterResult`. This breaks `isinstance` checks and mutable registry state.

The build system injects a one-liner to fix this:

```python
sys.modules.setdefault("dekube", sys.modules[__name__])
```

**Consequences for extension authors:**

- Prefer duck typing over `isinstance`. Use `getattr(result, 'services', None)` instead of `isinstance(result, ProviderResult)`.
- Import from `dekube`, not from internal submodules (`dekube.pacts.types`). The flat file has no submodules.
- The legacy `h2c` shim still works but new extensions should use `dekube`.

See [Build system — The `sys.modules` fix](understand/build-system.md#the-sysmodules-fix) for the full explanation.

## Build system gotchas

These affect distribution builders, not extension authors.

- **Artifact shadowing** — a `dekube.py` build artifact in the dev directory shadows the `src/dekube/` package. Delete it before running `PYTHONPATH=src python -m dekube`.
- **Module order** — `CORE_MODULES` in `build.py` must be topologically sorted. Wrong order → `NameError` on `--help` (the smoke test catches this).
- **Third-party detection** — the import sorter checks `module == "yaml"`. If a new third-party dep is added, update this check.
- **`_auto_register()` skips `_`-prefixed names** — naming a class `_MyConverter` prevents registration. New base classes must be added to `_BASE_CLASSES`.
- **Duplicate kind = fatal** — two classes claiming the same `kind` causes `sys.exit(1)`. Intentional — silent conflicts are worse.

See [Build system — Gotchas](understand/build-system.md#gotchas) for details on each.

## Docker/Compose gotchas

Runtime limitations of Docker Compose itself, not the conversion engine.

- **Large port ranges** — K8s with `hostNetwork` handles thousands of ports natively. Docker creates one iptables/pf rule per port, so a range like 50000-60000 (e.g. WebRTC) will kill your network stack. Reduce the range in your compose environment values.
- **hostNetwork** — every exposed port must be mapped explicitly in compose. K8s pods can bind directly; compose services cannot.
- **S3 virtual-hosted style** — AWS SDKs default to virtual-hosted bucket URLs (`bucket-name.s3:9000`). Compose DNS can't resolve dotted hostnames. Configure path-style access and use a `replacement` if needed.
