# Build system

> *The scribe's first task was not to write the scripture — it was to build the press. For the scripture was twenty-one tablets, and the faithful demanded a single scroll. The press crushes the tablets in order, strips the seams, and produces a scroll indistinguishable from one carved by hand. The scribe keeps both. The faithful see only the scroll. The inquisitors see everything.*
>
> — *Cultes des Goules, On the Machinery of Canon (notarized, allegedly)*

## Two scripts, two purposes

### `build.py` (in dekube-engine)

Concatenates `src/dekube/` into a single `dekube.py`. This is the bare engine — no extensions, no built-in converters, empty registries. Used to produce the dekube-engine release artifact.

```bash
cd dekube-engine && python build.py
# → dekube.py
```

### `build-distribution.py` (in dekube-engine, fetched from repo at build time)

Builds a distribution from core + extensions directory. Used by distribution repos (e.g. [helmfile2compose](https://github.com/dekubeio/helmfile2compose)) to produce a single-file script with bundled extensions.

```bash
# Local dev mode
cd helmfile2compose && python ../dekube-engine/build-distribution.py helmfile2compose \
  --extensions-dir extensions --core-dir ../dekube-engine
# → helmfile2compose.py

# CI mode (fetches dekube.py from GitHub release)
python build-distribution.py helmfile2compose --extensions-dir extensions
# → helmfile2compose.py
```

## How concatenation works

Both scripts follow the same core pattern:

1. **Module order**: Modules are concatenated in dependency order — `MODULES` (in `build.py`) / `CORE_MODULES` (in `build-distribution.py`). This list is manually maintained. If the order is wrong, the smoke test fails immediately.

2. **Internal imports stripped**: All `from dekube.*` imports are removed — everything's in one namespace after concatenation. Multi-line internal imports (parenthesized continuation) are handled.

3. **Stdlib/third-party imports deduplicated**: All `import` and `from ... import` statements are collected, deduplicated by text, and sorted (stdlib first, `yaml` second).

4. **Module docstrings stripped**: Triple-quoted docstrings at the top of each module are removed.

5. **Section comments**: Each module boundary is marked with `# --- core.env ---` (or similar) in the output for navigation.

6. **Smoke test**: After writing the output, both scripts run `python <output> --help` and fail if it exits non-zero.

## `build-distribution.py` specifics

### Three modes

- **`--core-dir` (local dev)**: Reads core sources from a local dekube-engine checkout. Processes each module in `CORE_MODULES` order, exactly like `build.py`.
- **CI against core (default)**: Fetches the flat `dekube.py` from the latest (or pinned) dekube-engine GitHub release. Strips its `__main__` guard, parses the already-concatenated imports + body, then appends extensions on top.
- **Stacking (`--base` or `--base-distribution`)**: Builds on top of another distribution instead of bare dekube-engine. `strip_tail()` removes the three tail blocks (`_auto_register()`, `sys.modules` hack, `__main__` guard) from the base before parsing. Extensions are appended, then the tail is re-generated. `--base` takes a local `.py` file; `--base-distribution` fetches from GitHub releases via dekube-manager's `distributions.json`.

### Extension discovery

Extensions are discovered from `--extensions-dir`: all `.py` files except `__init__.py` and hidden/underscore-prefixed files. Each is processed the same way as core modules (strip internal imports, collect stdlib imports, extract body).

### Post-concatenation steps

After all code (core + extensions), three incantations are appended. They must appear in this exact order, or the scroll is inert:

1. **`_auto_register()` call** — appended to populate converter/rewriter/transform registries from all classes in globals
2. **`sys.modules` hack** — registers the flat file as the `dekube` module so runtime-loaded extensions can `from dekube import ...`
3. **`__main__` guard** — `if __name__ == "__main__": main()`

## The `sys.modules` fix

Why it exists:

- When the output file is `dekube.py`, Python finds it natively as the `dekube` module. No hack needed. That's `build.py`.
- When the output file is `helmfile2compose.py` (or any other name), `from dekube import ...` fails because no `dekube` module exists on the import path.

The fix registers the running module as `dekube` in `sys.modules` before any extension can import:

```python
sys.modules.setdefault("dekube", sys.modules[__name__])
```

This matters for **both** runtime-loaded extensions (via `--extensions-dir`) and the `__main__` vs module identity problem. Without it, `isinstance` checks and mutable state (`_REWRITERS`, `_CONVERTERS`) break because `__main__.ProviderResult` ≠ `dekube.ProviderResult`. The `setdefault` ensures extensions always resolve to the running module instance — no snapshot, no copy, no split identity.

## Gotchas

### 1. Artifact shadowing

If `dekube.py` (build artifact) exists in the current directory while running `PYTHONPATH=src python -m dekube`, Python finds the flat file instead of the package. Error: `'dekube' is not a package`.

**Fix:** Delete the artifact, or don't build in the same directory you develop from.

### 2. Module order matters

`CORE_MODULES` must be topological. A function defined in `core/env.py` that uses something from `pacts/helpers.py` requires `pacts/helpers.py` to appear earlier in the list. The smoke test catches this immediately — if the order is wrong, `--help` crashes on a `NameError`.

### 3. Third-party detection is naive

The import sorter checks `module == "yaml"` to identify third-party packages. If a new third-party dependency is added, update this check in both build scripts. Currently only `pyyaml` exists as a third-party dep.

### 4. `_auto_register()` skips base classes and `_`-prefixed names

If you name an extension class `_MyConverter`, it won't be registered. If you create a new base class, add it to `_BASE_CLASSES` in `core/convert.py`.

### 5. Duplicate kind detection is fatal

`_auto_register()` calls `sys.exit(1)` if two classes claim the same kind. This is intentional — silent conflicts are worse. If you see:

```
Error: kind 'Ingress' claimed by both CaddyProvider and MyIngressProvider
```

...one of them needs to go. A distribution can only have one handler per kind.

## Running locally

```bash
# Build bare core
cd dekube-engine && python build.py
# → dekube.py

# Build distribution (local dev)
cd helmfile2compose && python ../dekube-engine/build-distribution.py helmfile2compose \
  --extensions-dir extensions --core-dir ../dekube-engine
# → helmfile2compose.py

# Build distribution (CI mode, fetches from release)
python build-distribution.py helmfile2compose --extensions-dir extensions

# Build stacked distribution (local base)
cd kubernetes2simple && python ../dekube-engine/build-distribution.py kubernetes2simple \
  --extensions-dir .dekube/extensions --base ../helmfile2compose/helmfile2compose.py
# → kubernetes2simple.py

# Build stacked distribution (CI mode, fetches base from release)
python build-distribution.py kubernetes2simple \
  --extensions-dir .dekube/extensions --base-distribution helmfile2compose
```
