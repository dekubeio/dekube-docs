# Distributions

> *The high priest declared the engine too pure for mortal use, for it carried no doctrine, no creed, no opinion. And so he dressed it in vestments — converters for the faithful, rewriters for the heretics, indexers for the bureaucrats — and called the result a "distribution." The engine did not object. It had no opinions. That was the point.*
>
> — *Book of Eibon, On the Dressing of Engines (verbatim, more or less)*

## What

A distribution is h2c-core (the bare engine) + a set of bundled extensions, concatenated into a single Python script. The engine provides the pipeline, CLI, and extension loader; the distribution decides what's built in.

If you're reading this and thinking "this is the Kubernetes distribution model" — yes. A bare apiserver that does nothing without controllers, wrapped by a distribution (k3s, EKS, etc.) that bundles the defaults. I arrived here by solving engineering problems one at a time, each decision locally reasonable, and looked up to find I had recreated the architecture of the very thing I was trying to bring to the uninitiated. The concepts page has a [longer meditation on this](concepts.md#the-ouroboros) for those who enjoy watching someone realize what they've done.

```
h2c-core (engine)  +  extensions  =  distribution
    h2c.py                              helmfile2compose.py

                 stacking
helmfile2compose  +  more extensions  =  kubernetes2simple.py
```

Distributions can be **stacked**: built on top of another distribution instead of bare h2c-core. `kubernetes2simple` stacks on `helmfile2compose`, adding all official extensions. The build script strips the tail (registries, `sys.modules` hack, `__main__` guard) from the base, appends new extensions, and re-appends the tail.

## Why

Different organizations want different defaults. The helmfile2compose distribution bundles Caddy as the reverse proxy, HAProxy as the default rewriter, and WorkloadConverter for standard K8s kinds. A corporate distribution might bundle Traefik instead, add custom CRD handlers, or remove features that don't apply.

The distribution model avoids forking the core — everyone shares the same engine, just with different extensions wired in.

## helmfile2compose — the reference distribution

[helmfile2compose](https://github.com/helmfile2compose/helmfile2compose) is the default distribution. It bundles 7 extensions:

| Future repo | Type | File in `extensions/` | Purpose |
|-------------|------|-----------------------|---------|
| h2c-indexer-configmap | IndexerConverter | `configmap_indexer.py` | Populates `ctx.configmaps` |
| h2c-indexer-secret | IndexerConverter | `secret_indexer.py` | Populates `ctx.secrets` |
| h2c-indexer-pvc | IndexerConverter | `pvc_indexer.py` | Populates `ctx.pvc_names` |
| h2c-indexer-service | IndexerConverter | `service_indexer.py` | Populates `ctx.services_by_selector` |
| h2c-converter-workload | Provider | `workloads.py` | DaemonSet, Deployment, Job, StatefulSet → compose services |
| h2c-rewriter-haproxy | IngressRewriter | `haproxy.py` | HAProxy annotations + default fallback |
| h2c-provider-caddy | IngressProvider | `caddy.py` | Caddy service + Caddyfile generation |

External extensions (loaded via `--extensions-dir` at runtime) work on top of whatever a distribution bundles.

## Distribution repo structure

```
my-distribution/
├── extensions/
│   ├── __init__.py           # empty (required for discovery to skip)
│   ├── workloads.py          # WorkloadConverter
│   ├── haproxy.py            # HAProxyRewriter
│   ├── caddy.py              # CaddyProvider
│   ├── configmap_indexer.py   # IndexerConverter
│   ├── secret_indexer.py      # IndexerConverter
│   ├── pvc_indexer.py         # IndexerConverter
│   └── service_indexer.py     # IndexerConverter
├── .github/workflows/
│   └── release.yml           # CI: fetch h2c.py + build distribution
└── README.md
```

Extensions in `extensions/` are `.py` files — each discovered automatically by `build-distribution.py`. `__init__.py` and hidden files are skipped.

## `_auto_register()` — how it works

After all code (core + extensions) is concatenated, the build script appends a call to `_auto_register()`. This function:

1. Scans the caller's globals for classes
2. Skips base classes (`Converter`, `Provider`, `IndexerConverter`, `IngressProvider`, `IngressRewriter`) and `_`-prefixed names
3. Identifies converters (have `kinds` + `convert()`), rewriters (have `name` + `match()` + `rewrite()`), and transforms (have `transform()`, no `kinds`)
4. Checks for duplicate kind claims — **fatal** if two classes claim the same kind (`sys.exit(1)`)
5. Sorts by priority and populates `_CONVERTERS`, `_REWRITERS`, `_TRANSFORMS`, `CONVERTED_KINDS`

This means you just define classes in your extension files — no manual registration, no registry boilerplate.

## Stacking distributions

A distribution can be built on top of another distribution instead of bare h2c-core. The `--base-distribution` flag tells `build-distribution.py` to fetch the parent from GitHub releases (resolved via h2c-manager's `distributions.json`), strip its tail blocks, and use it as the base instead of `h2c.py`.

```bash
# Build kubernetes2simple on top of helmfile2compose
python build-distribution.py kubernetes2simple \
  --extensions-dir .h2c/extensions \
  --base-distribution helmfile2compose

# Or use a local .py file as base
python build-distribution.py kubernetes2simple \
  --extensions-dir .h2c/extensions \
  --base ../helmfile2compose/helmfile2compose.py
```

The `--base-distribution` flag defaults to `core` (bare h2c-core). When set to another distribution name, `build-distribution.py` looks it up in h2c-manager's `distributions.json`, downloads the `.py` release artifact, and uses it as the base. `--base-version` pins a specific version (default: `latest`).

`--base` takes a local `.py` file path directly — useful for local dev when you have the parent distribution already built.

## CI workflow example

### Standard distribution (against h2c-core)

```yaml
name: Release
on:
  release:
    types: [published]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Fetch build-distribution.py
        run: |
          curl -fsSL https://raw.githubusercontent.com/helmfile2compose/h2c-core/main/build-distribution.py \
            -o build-distribution.py

      - name: Build distribution
        run: python3 build-distribution.py helmfile2compose --extensions-dir extensions

      - name: Upload release assets
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh release upload "${{ github.event.release.tag_name }}" helmfile2compose.py
```

### Stacked distribution (against another distribution)

```yaml
- name: Build kubernetes2simple.py
  run: |
    python build-distribution.py kubernetes2simple \
      --extensions-dir .h2c/extensions \
      --base-distribution helmfile2compose
```

## Dev workflow

```bash
# Build locally (reads core sources from sibling checkout)
cd my-distribution
python ../h2c-core/build-distribution.py helmfile2compose \
  --extensions-dir extensions --core-dir ../h2c-core
# → helmfile2compose.py

# Build stacked distribution locally
cd kubernetes2simple
python ../h2c-core/build-distribution.py kubernetes2simple \
  --extensions-dir .h2c/extensions \
  --base ../helmfile2compose/helmfile2compose.py
# → kubernetes2simple.py

# Test it
python helmfile2compose.py --from-dir /tmp/rendered --output-dir /tmp/out
```

See [Build system](build-system.md) for the full deep dive on how concatenation works, the `sys.modules` hack, and all the gotchas.
