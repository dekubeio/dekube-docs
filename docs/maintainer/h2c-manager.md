# h2c-manager

h2c-manager is a lightweight package manager for helmfile2compose. It downloads the core script and extensions from GitHub releases. Python 3, stdlib only — no dependencies.

## Installation

Download from the h2c-manager repo (rolling release — always latest):

```bash
curl -fsSL https://raw.githubusercontent.com/helmfile2compose/h2c-manager/main/h2c-manager.py -o h2c-manager.py
```

## Usage

```bash
# Core only (no extensions)
python3 h2c-manager.py

# Core + keycloak extension
python3 h2c-manager.py keycloak

# Core + multiple extensions
python3 h2c-manager.py keycloak cert-manager trust-manager

# Pin core version
python3 h2c-manager.py --core-version v2.1.0 keycloak

# Pin extension version
python3 h2c-manager.py keycloak==0.2.0

# Custom install directory (default: .h2c/)
python3 h2c-manager.py -d ./tools keycloak

# Skip download, reuse cached .h2c/ (missing files still downloaded)
python3 h2c-manager.py --no-reinstall

# Same with run mode
python3 h2c-manager.py --no-reinstall run -e compose
```

## Output

h2c-manager creates the following directory structure:

```
.h2c/
├── helmfile2compose.py       # core script
└── extensions/
    ├── keycloak.py            # requested extension
    └── cert_manager.py         # auto-resolved dependency
```

## Run mode

`run` is a shortcut to invoke helmfile2compose with sensible defaults:

```bash
python3 h2c-manager.py run -e compose
# equivalent to:
# python3 .h2c/helmfile2compose.py --helmfile-dir . --extensions-dir .h2c/extensions --output-dir . -e compose
```

By default, `run` re-downloads h2c-core and all extensions before every invocation, overwriting cached files. Versions follow normal resolution: latest release, or the pinned version from `helmfile2compose.yaml` / CLI flags. If you change a pin in the yaml, the next `run` picks it up.

Use `--no-reinstall` to skip downloads for files already present in `.h2c/` (missing files are still fetched). There is no version tracking: a cached file is either kept as-is or replaced with whatever version resolves. No in-between.

Defaults: `--helmfile-dir .`, `--extensions-dir .h2c/extensions` (if it exists), `--output-dir .`. Any explicit flag overrides the default. All extra arguments are passed through to helmfile2compose.

## Version resolution

### Core

- No flag: latest GitHub release of `helmfile2compose/h2c-core`
- `--core-version v2.1.0`: exact tag

### Extensions

- Bare name (`keycloak`): latest GitHub release
- Pinned (`keycloak==0.2.0`): exact version. The `v` prefix is added automatically if missing (`0.1.0` -> `v0.1.0`).

No version ranges. Just `latest` and `==exact`.

## Dependency resolution

Extensions can declare dependencies in the registry. For example, `trust-manager` depends on `cert-manager`:

```bash
python3 h2c-manager.py trust-manager
# Fetching h2c-core v2.1.0...
# Fetching extension cert-manager v0.1.0 (dependency of trust-manager)...
# Fetching extension trust-manager v0.1.1...
```

Dependencies are resolved one level deep (no transitive chains). Duplicates are deduplicated.

## Incompatibility checking

Some extensions are fundamentally incompatible (e.g. `flatten-internal-urls` strips FQDNs that `cert-manager` certificates rely on). The registry declares these conflicts via `"incompatible"` in `extensions.json`, and h2c-manager blocks the combination before downloading anything:

```
Error: extensions 'flatten-internal-urls' and 'cert-manager' are incompatible
  Use --ignore-compatibility-errors flatten-internal-urls to override
```

The check is bidirectional — if A declares incompatibility with B, installing both A+B or B+A triggers the error.

To override (you know what you're doing):

```bash
# Install mode
python3 h2c-manager.py cert-manager flatten-internal-urls --ignore-compatibility-errors flatten-internal-urls

# Run mode
python3 h2c-manager.py run -e compose --ignore-compatibility-errors flatten-internal-urls
```

## Python dependency checking

Some extensions require Python packages (e.g. cert-manager requires `cryptography`). h2c-manager checks if they're installed and prints an aggregated warning:

```
Missing Python dependencies:
  cert-manager: cryptography
Install with: pip install cryptography
```

This is a warning, not a fatal error — the extension file is still downloaded. Install the dependency before running helmfile2compose with that extension.

## Extension registry

The registry lives at `helmfile2compose/h2c-manager` as `extensions.json`. It maps extension names to GitHub repos:

```json
{
  "schema_version": 1,
  "extensions": {
    "keycloak": {
      "repo": "helmfile2compose/h2c-operator-keycloak",
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

See [Writing operators — publishing](../developer/writing-operators.md#publishing) for the full guide.

## Declarative dependencies

If `helmfile2compose.yaml` exists, h2c-manager reads `core_version` and `depends` from it:

```yaml
# helmfile2compose.yaml
core_version: v2.1.0
depends:
  - keycloak
  - cert-manager==0.1.0
  - trust-manager
```

```bash
python3 h2c-manager.py
# Core version from helmfile2compose.yaml: v2.1.0
# Fetching h2c-core v2.1.0...
# Reading extensions from helmfile2compose.yaml: keycloak, cert-manager==0.1.0, trust-manager
# Fetching extension keycloak v0.2.0...
# ...
```

CLI flags override the yaml: `--core-version` overrides `core_version`, explicit extension args override `depends`. This keeps behavior predictable: either the yaml drives it, or you drive it.

h2c-manager also checks that `pyyaml` is installed (required by h2c-core) and warns alongside any extension-specific missing dependencies.

## Integration with generate-compose.sh

Projects that ship a `generate-compose.sh` wrapper can use h2c-manager as the bootstrap:

```bash
#!/bin/bash
set -euo pipefail

H2C_MANAGER_URL="https://raw.githubusercontent.com/helmfile2compose/h2c-manager/main/h2c-manager.py"
curl -fsSL "$H2C_MANAGER_URL" -o h2c-manager.py
python3 h2c-manager.py          # reads depends from helmfile2compose.yaml
python3 h2c-manager.py run -e compose
```
