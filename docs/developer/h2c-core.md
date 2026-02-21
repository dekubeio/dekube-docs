# h2c-core — the bare engine

> *The disciple gazed upon the monolith and saw that it was vast — a single tablet bearing every law, every rite, every contradiction. And he said: let us shatter the tablet, that each fragment may be understood alone. But when the pieces lay scattered, each fragment still whispered the name of the whole.*
>
> — *Necronomicon, On the Shattering of Tablets (fragmentary)*

## What it is

h2c-core is the conversion engine — the pipeline, the extension loader, the CLI, and nothing else. No built-in converters. No built-in rewriters. All registries empty. Feed it manifests and it will parse them, warn that every kind is unknown, and produce nothing. A temple with no priests.

It exists so that [distributions](distributions.md) can bundle different sets of extensions on top of the same engine. The [helmfile2compose](https://github.com/helmfile2compose/helmfile2compose) distribution is the default — h2c-core + 7 built-in extensions, concatenated into a single `helmfile2compose.py`. But h2c-core can also be used standalone with `--extensions-dir`, or as the foundation for custom distributions.

**Users never interact with h2c-core directly.** They use a distribution. h2c-core is for extension developers, distribution builders, and people who want to understand how the engine works.

## Package structure

```
h2c-core/
├── src/h2c/
│   ├── __init__.py          # re-exports pacts API
│   ├── __main__.py          # python -m h2c entry point
│   ├── cli.py               # argument parsing, orchestration
│   ├── pacts/               # public contracts for extensions
│   │   ├── types.py         # ConvertContext, ConvertResult, Converter, Provider
│   │   ├── ingress.py       # IngressRewriter, get_ingress_class(), resolve_backend()
│   │   └── helpers.py       # apply_replacements(), _secret_value()
│   ├── core/                # internal conversion engine
│   │   ├── constants.py     # regexes, kind lists, well-known ports
│   │   ├── env.py           # resolve_env(), port remapping, command conversion
│   │   ├── volumes.py       # volume mount conversion (PVC, ConfigMap, Secret)
│   │   ├── services.py      # K8s Service indexing, alias maps, port maps
│   │   ├── ingress.py       # IngressProvider, rewriter dispatch, _NullRewriter
│   │   ├── extensions.py    # extension discovery and loading
│   │   └── convert.py       # convert() orchestration, overrides, fix-permissions, _auto_register()
│   └── io/                  # input/output
│       ├── parsing.py       # helmfile template, YAML loading, namespace inference
│       ├── config.py        # helmfile2compose.yaml load/save
│       └── output.py        # compose.yml, warnings
├── build.py                 # concat → single-file h2c.py (bare engine)
├── build-distribution.py    # concat core + extensions → distribution
└── .github/workflows/
    └── release.yml          # CI: build + release assets on tag push
```

## The three layers

### `pacts/` — the sacred contracts {#pacts--the-sacred-contracts}

Everything extensions can import. This is the stable API:

```python
from h2c import ConvertContext, ConvertResult
from h2c import Converter, IndexerConverter, Provider
from h2c import IngressRewriter, get_ingress_class, resolve_backend
from h2c import apply_replacements, resolve_env, _secret_value
```

Or explicitly:

```python
from h2c.pacts import ConvertContext, IngressRewriter
from h2c.pacts.types import Provider
```

Both paths work — `__init__.py` re-exports the pacts API. These are the only imports extensions should use. If it's not in `pacts/`, it's internal and may change.

### `core/` — the conversion engine

Internal modules that do the actual work. Not part of the public API. The dependency graph is acyclic:

```
constants.py      ← standalone
env.py            ← pacts/helpers, constants
volumes.py        ← pacts/helpers, env
services.py       ← constants
ingress.py        ← pacts, IngressProvider base class
extensions.py     ← pacts/ingress, ingress
convert.py        ← pacts, all core modules, _auto_register()
```

Module-level mutable state lives in `core/convert.py` (`_CONVERTERS`, `_TRANSFORMS`, `CONVERTED_KINDS`) and `core/ingress.py` (`_REWRITERS`). In the distribution model, `_auto_register()` populates these registries from all classes found in the concatenated script's globals.

### `io/` — input/output

Parsing, config, and output writing. No conversion logic — just plumbing between the filesystem and the core.

## Build artifacts

Two build scripts produce two different outputs:

- **`build.py`** — concatenates `src/h2c/` into a single `h2c.py`. Bare engine, no extensions. This is the h2c-core release artifact.
- **`build-distribution.py`** — builds a distribution from core + extensions dir. Used by distribution repos. Published as a release asset so distributions can fetch it in CI.

```bash
# Bare core
python build.py
# → h2c.py (~1265 lines, not committed)

# Distribution (from a distribution repo)
python build-distribution.py helmfile2compose --extensions-dir extensions --core-dir ../h2c-core
# → helmfile2compose.py
```

See [Build system](build-system.md) for the full deep dive on concatenation, import stripping, and the `sys.modules` hack.

## Dev workflow

```bash
# Run from the package (development)
PYTHONPATH=src python -m h2c --from-dir /tmp/rendered --output-dir /tmp/out

# Build and smoke test
python build.py
python h2c.py --from-dir /tmp/rendered --output-dir /tmp/out
```
