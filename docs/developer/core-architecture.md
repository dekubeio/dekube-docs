# Core architecture

> *The disciple gazed upon the monolith and saw that it was vast — a single tablet bearing every law, every rite, every contradiction. And he said: let us shatter the tablet, that each fragment may be understood alone. But when the pieces lay scattered, each fragment still whispered the name of the whole.*
>
> — *Necronomicon, On the Shattering of Tablets (fragmentary)*

## Why split

`helmfile2compose.py` grew to ~1860 lines — a single file containing types, constants, parsers, converters, CLI, and output writers. It worked, but navigating it required a mental index. The split preserves the single-file distribution (users and h2c-manager see no change) while making the codebase navigable.

## Package structure

```
h2c-core/
├── src/helmfile2compose/
│   ├── __init__.py          # re-exports pacts API (backwards compat)
│   ├── __main__.py          # python -m helmfile2compose entry point
│   ├── cli.py               # argument parsing, orchestration
│   ├── pacts/               # public contracts for extensions
│   │   ├── __init__.py
│   │   ├── types.py         # ConvertContext, ConvertResult
│   │   ├── ingress.py       # IngressRewriter, get_ingress_class(), resolve_backend()
│   │   └── helpers.py       # apply_replacements(), resolve_env(), _secret_value()
│   ├── core/                # internal conversion engine
│   │   ├── __init__.py
│   │   ├── constants.py     # regexes, kind lists, well-known ports
│   │   ├── env.py           # env resolution, port remapping, command conversion
│   │   ├── volumes.py       # volume mount conversion (PVC, ConfigMap, Secret)
│   │   ├── services.py      # K8s Service indexing, alias maps, port maps
│   │   ├── workloads.py     # WorkloadConverter + init/sidecar containers
│   │   ├── haproxy.py       # HAProxyRewriter (built-in, structurally like an extension)
│   │   ├── ingress.py       # IngressConverter, rewriter dispatch, _NullRewriter
│   │   ├── extensions.py    # extension discovery and loading
│   │   └── convert.py       # convert() orchestration, overrides, fix-permissions
│   └── io/                  # input/output
│       ├── __init__.py
│       ├── parsing.py       # helmfile template, YAML loading, namespace inference
│       ├── config.py        # helmfile2compose.yaml load/save
│       └── output.py        # compose.yml, Caddyfile, warnings
├── build.py                 # concat → single-file helmfile2compose.py
└── .github/workflows/
    └── release.yml          # CI: build + release asset on tag push
```

## The three layers

### `pacts/` — the sacred contracts {#pacts--the-sacred-contracts}

Everything extensions can import. This is the stable API:

```python
from helmfile2compose import ConvertContext, ConvertResult
from helmfile2compose import IngressRewriter, get_ingress_class, resolve_backend
from helmfile2compose import apply_replacements, resolve_env
```

Or explicitly:

```python
from helmfile2compose.pacts import ConvertContext, IngressRewriter
```

Both paths work — `__init__.py` re-exports the pacts API. These are the only imports extensions should use. If it's not in `pacts/`, it's internal and may change.

### `core/` — the conversion engine

Internal modules that do the actual work. Not part of the public API. The dependency graph is acyclic:

```
constants.py      ← standalone
env.py            ← pacts/helpers, constants
volumes.py        ← pacts/helpers, env
services.py       ← constants, workloads (one lazy import)
workloads.py      ← pacts, env, volumes
haproxy.py        ← pacts/ingress
ingress.py        ← pacts, constants, haproxy
extensions.py     ← pacts/ingress, ingress
convert.py        ← pacts, all core modules
```

Module-level mutable state lives in `core/convert.py` (`_CONVERTERS`, `_TRANSFORMS`, `CONVERTED_KINDS`) and `core/ingress.py` (`_REWRITERS`). Extensions mutate these registries at load time via `_register_extensions()`.

### `io/` — input/output

Parsing, config, and output writing. No conversion logic — just plumbing between the filesystem and the core.

## Build / concat

The single-file distribution is a build artifact. `build.py` concatenates the package:

1. Reads modules in manually-maintained dependency order (constants first, CLI last) — the `MODULES` list in `build.py` must respect the import graph. A topological sort would automate this, but 15 lines that change once a year don't warrant a DAG resolver. If the order is wrong, the `--help` smoke test fails immediately.
2. Strips internal `from helmfile2compose...` imports (everything's in one namespace)
3. Deduplicates stdlib/external imports
4. Writes `helmfile2compose.py` with shebang + docstring + all code
5. Runs `--help` as a smoke test

```bash
python build.py
# → helmfile2compose.py (build artifact, not committed)
```

The CI workflow (`.github/workflows/release.yml`) runs `build.py` on tag push and uploads `helmfile2compose.py` as a release asset.

## Dev workflow

```bash
# Run from the package (development)
cd h2c-core
PYTHONPATH=src python -m helmfile2compose --from-dir /tmp/rendered --output-dir /tmp/out

# Build the single-file version (testing distribution)
python build.py
python helmfile2compose.py --from-dir /tmp/rendered --output-dir /tmp/out

# Validate with the executioner
cd ../h2c-testsuite
./run-tests.sh --local-core ../h2c-core/helmfile2compose.py
```

Both paths produce identical output — the executioner validates this.
