# dekube-manager

dekube-manager is a lightweight package manager for helmfile2compose. It downloads the distribution script and extensions from GitHub releases. Python 3, stdlib only — no dependencies.

## Installation

Download from the dekube-manager repo (rolling release — always latest):

```bash
curl -fsSL https://raw.githubusercontent.com/dekubeio/dekube-manager/main/dekube-manager.py -o dekube-manager.py
```

## Usage

```bash
# Distribution only (no extensions)
python3 dekube-manager.py

# Distribution + keycloak extension
python3 dekube-manager.py keycloak

# Distribution + multiple extensions
python3 dekube-manager.py keycloak cert-manager trust-manager

# Pin distribution version
python3 dekube-manager.py --distribution-version v3.1.0 keycloak

# Pin extension version
python3 dekube-manager.py keycloak==0.4.0

# Custom install directory (default: .dekube/)
python3 dekube-manager.py -d ./tools keycloak

# Skip download, reuse cached .dekube/ (missing files still downloaded)
python3 dekube-manager.py --no-reinstall

# Same with run mode
python3 dekube-manager.py --no-reinstall run -e compose
```

## Info mode

`--info` displays registry information about extensions without downloading anything:

```bash
# Show info for all registered extensions
python3 dekube-manager.py --info

# Show info for specific extensions (includes dependencies)
python3 dekube-manager.py --info nginx traefik
```

If no extension names are given and `helmfile2compose.yaml` has a `depends:` list, info is shown for those extensions.

## Output

dekube-manager creates the following directory structure:

```
.dekube/
├── helmfile2compose.py       # distribution script
└── extensions/
    ├── keycloak.py            # requested extension
    └── cert_manager.py         # auto-resolved dependency
```

## Run mode

`run` is a shortcut to invoke helmfile2compose with sensible defaults:

```bash
python3 dekube-manager.py run -e compose
# equivalent to:
# python3 .dekube/helmfile2compose.py --helmfile-dir . --extensions-dir .dekube/extensions --output-dir . -e compose
```

By default, `run` re-downloads the distribution and all extensions before every invocation, overwriting cached files. Versions follow normal resolution: latest release, or the pinned version from `helmfile2compose.yaml` / CLI flags. If you change a pin in the yaml, the next `run` picks it up.

Use `--no-reinstall` to skip downloads for files already present in `.dekube/` (missing files are still fetched). There is no version tracking: a cached file is either kept as-is or replaced with whatever version resolves. No in-between.

Defaults: `--helmfile-dir .`, `--extensions-dir .dekube/extensions` (if it exists), `--output-dir .`. Any explicit flag overrides the default. All extra arguments are passed through to helmfile2compose.

**Note:** In `run` mode, extensions are resolved from `helmfile2compose.yaml` `depends:` (or CLI extension args if provided). Extension names passed *before* the `run` keyword are ignored — only `depends:` and post-`run` flags matter.

## Version resolution

### Distribution

- No flag: latest GitHub release of `dekubeio/helmfile2compose`
- `--distribution-version v3.1.0`: exact tag. The `v` prefix is added automatically if missing.

### Extensions

- Bare name (`keycloak`): latest GitHub release
- Pinned (`keycloak==0.4.0`): exact version. The `v` prefix is added automatically if missing (`0.1.0` -> `v0.1.0`).

No version ranges. Just `latest` and `==exact`.

## Dependency resolution

Extensions can declare dependencies in the registry. For example, `trust-manager` depends on `cert-manager`:

```bash
python3 dekube-manager.py trust-manager
# Fetching helmfile2compose (latest)...
# Fetching extension cert-manager (dependency of trust-manager)...
# Fetching extension trust-manager...
```

Dependencies are resolved one level deep (no transitive chains). Duplicates are deduplicated.

## Incompatibility checking

Some extensions are fundamentally incompatible (e.g. `cert-manager` generates certificates with FQDN SANs that `flatten-internal-urls` would strip). The registry declares these conflicts via `"incompatible"` in `extensions.json`, and dekube-manager blocks the combination before downloading anything:

```
Error: extensions 'cert-manager' and 'flatten-internal-urls' are incompatible
  Use --ignore-compatibility-errors cert-manager to override
```

The check is bidirectional — if A declares incompatibility with B, installing both A+B or B+A triggers the error.

To override (you know what you're doing):

```bash
# Install mode
python3 dekube-manager.py cert-manager flatten-internal-urls --ignore-compatibility-errors cert-manager

# Run mode
python3 dekube-manager.py run -e compose --ignore-compatibility-errors cert-manager
```

## Python dependency checking

Some extensions require Python packages (e.g. cert-manager requires `cryptography`). dekube-manager checks if they're installed and prints an aggregated warning:

```
Missing Python dependencies:
  cert-manager: cryptography
Install with: pip install cryptography
```

This is a warning, not a fatal error — the extension file is still downloaded. Install the dependency before running helmfile2compose with that extension.

## Extension registry

The registry lives at `dekubeio/dekube-manager` as `extensions.json`. It maps extension names to GitHub repos:

```json
{
  "schema_version": 1,
  "extensions": {
    "keycloak": {
      "repo": "dekubeio/dekube-provider-keycloak",
      "description": "Keycloak and KeycloakRealmImport CRDs",
      "file": "keycloak.py",
      "depends": [],
      "incompatible": []
    }
  }
}
```

Available versions are whatever tags/releases exist on each repo. The registry doesn't list versions — GitHub is the source of truth.

### Adding an extension to the registry

See [Writing extensions — publishing](../developer/extensions/index.md#publishing) for the full guide.

## Declarative dependencies

If `helmfile2compose.yaml` exists, dekube-manager reads `distribution`, `distribution_version`, and `depends` from it:

```yaml
# helmfile2compose.yaml
distribution: helmfile2compose    # optional — default is helmfile2compose
distribution_version: v3.1.0
depends:
  - keycloak
  - cert-manager==0.3.0
  - trust-manager
```

The `distribution` key selects which distribution to install. Available distributions are listed in the `distributions.json` registry. Use `core` for the bare engine (`dekube.py`) or omit the key for the default full distribution.

```bash
python3 dekube-manager.py
# Distribution version from helmfile2compose.yaml: v3.1.0
# Fetching helmfile2compose v3.1.0...
# Reading extensions from helmfile2compose.yaml: keycloak, cert-manager==0.3.0, trust-manager
# Fetching extension keycloak v0.4.0...
# ...
```

CLI flags override the yaml: `--distribution-version` overrides `distribution_version`, explicit extension args override `depends`. This keeps behavior predictable: either the yaml drives it, or you drive it.

`core_version` is accepted as a backwards-compatible alias for `distribution_version`. The `v` prefix is added automatically if missing (`3.0.0` → `v3.0.0`).

dekube-manager also checks that `pyyaml` is installed (required by helmfile2compose) and warns alongside any extension-specific missing dependencies.

## Integration with generate-compose.sh

Projects that ship a `generate-compose.sh` wrapper can use dekube-manager as the bootstrap:

```bash
#!/bin/bash
set -euo pipefail

H2C_MANAGER_URL="https://raw.githubusercontent.com/dekubeio/dekube-manager/main/dekube-manager.py"
curl -fsSL "$H2C_MANAGER_URL" -o dekube-manager.py
python3 dekube-manager.py run -e compose
```
