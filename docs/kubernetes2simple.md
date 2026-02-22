# kubernetes2simple

> *The disciple came to the temple seeking passage. "I wish to cross," he said, "but I do not wish to understand." The priests exchanged a glance. They had built an entire wing for this.*
>
> — *Necronomicon, On Blessed Ignorance (understandably)*

*Kubernetes? Too simple.*

Also known as **k2s** — not to be confused with [K2s](https://github.com/Siemens-Healthineers/K2s) (uppercase), the Siemens Healthineers Kubernetes distribution for Windows. Different project, same cosmic arrogance.

You have a Kubernetes project — a helmfile, a Helm chart, or just a pile of YAML. You want `docker compose up -d`. You don't want to install anything, configure anything, or learn anything.

```bash
curl -fsSL https://raw.githubusercontent.com/helmfile2compose/kubernetes2simple/main/kubernetes2simple.sh -o k2s.sh
chmod +x k2s.sh
./k2s.sh
docker compose up -d
```

That's it.

## What happens

The script looks at your current directory and figures out what you have:

| It finds | It does |
|---|---|
| `helmfile.yaml` | Renders via helmfile, then converts |
| `Chart.yaml` | Renders via helm, then converts |
| `*.yaml` with K8s manifests | Converts directly |
| Nothing recognizable | Tells you, politely |

Then it installs whatever's missing — Python dependencies, helm, helmfile — into a `.kubernetes2simple/` directory that stays in your project. Your system is never touched. Add `.kubernetes2simple/` to your `.gitignore` and forget about it.

The output is a `compose.yml`, a `Caddyfile`, and a `helmfile2compose.yaml` configuration file.

## A few things worth knowing

**You can re-run safely.** `compose.yml` and `Caddyfile` are regenerated every time — don't edit them by hand. To customize things (exclude a service, override an image, pin a volume path), edit `helmfile2compose.yaml`. That file is yours; the script reads it but never overwrites it.

**TLS is best-effort.** If your project uses cert-manager, the script generates self-signed certificates locally so services can start. This is fine for development. It is not real TLS — don't ship it expecting production certificates.

**Some things won't convert.** CronJobs, resource limits, HPA, probes — anything without a compose equivalent is skipped with a warning. The goal is a working local environment, not a replica of your cluster.

## Options

```
./k2s.sh [--env <environment>] [--output-dir <dir>] [--clean]
```

| Flag | What it does |
|---|---|
| `--env`, `-e` | Helmfile environment to render (helmfile mode only) |
| `--output-dir` | Where to write the output (default: current directory) |
| `--clean` | Wipe `.kubernetes2simple/` and start fresh |

## Prerequisites

- Docker (with `docker compose`)
- Python 3.10+
- curl
- Internet access on first run

Everything else is handled.

## It didn't work

[Open an issue.](https://github.com/helmfile2compose/kubernetes2simple/issues) Paste the output. We'll figure it out.

---

## Into the abyss

kubernetes2simple is the surface. Underneath is [helmfile2compose](index.md) — a conversion engine with an extension system, a package manager, a fake API server, a regression suite with a torture generator, and documentation that you are currently reading.

If you want to understand what just happened to your YAML, you can start here:

- **[What gets converted and how](developer/architecture.md)** — the conversion pipeline, kind by kind
- **[Configuration deep dive](maintainer/configuration.md)** — everything `helmfile2compose.yaml` can do
- **[Available extensions](catalogue.md)** — Keycloak, cert-manager, Prometheus, Bitnami workarounds, and more
- **[Limitations](limitations.md)** — what doesn't cross the bridge, and why
- **[About](about.md)** — the full, unhinged story of how this happened

You were warned.
