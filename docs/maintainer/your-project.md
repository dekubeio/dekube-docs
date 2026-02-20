# Using helmfile2compose with your own project

I'm sorry you're here. Truly. If you're reading this, it means you maintain a helmfile (or at least Helm charts), and someone — a colleague, a client, a mass of disciples — has asked you for a docker-compose. You have exhausted all diplomatic avenues. You have explained that Kubernetes exists for a reason. They do not care. They want `docker compose up` and they want it yesterday.

I've been there. Twice. This tool is the scar tissue.

From now on we will call this project **h2c**, because the concept is already mind-numbing enough — reading the full "helmfile2compose" every three words will certainly not help.

> *He who renders the celestial into the mundane does not ascend — he merely ensures that both realms now share his suffering equally.*
>
> — *Necronomicon, On the Folly of Downward Translation (I think)*

For the dark and twisted ritual underlying the conversion — what gets converted, how, and what unholy transformations are applied — see [Architecture](../developer/architecture.md).

## Requirements

- Python 3.10+
- `pyyaml`
- `helmfile` + `helm` (only if rendering from helmfile directly)
- **Docker Compose** (v2) — recommended for running the generated output. nerdctl compose requires the `flatten-internal-urls` transform to work (see [Network aliases](../limitations.md#network-aliases-nerdctl)). Podman Compose works (v1.0.6+).

## Before you start: ingress controller

Your helmfile already uses an ingress controller. That choice is baked into your Ingress manifests — annotations, `ingressClassName` — and you're not going to change it for a compose migration. h2c needs a **rewriter** that speaks your controller's annotation dialect and translates it to Caddy. If your controller isn't supported, your Ingress manifests will be silently skipped and you'll have no reverse proxy — which is the single most visible thing that breaks.

| Controller | Rewriter | Status |
|------------|----------|--------|
| **HAProxy** | built-in | stable |
| **Nginx** | [h2c-rewriter-nginx](https://github.com/helmfile2compose/h2c-rewriter-nginx) (extension) | stable |
| **Traefik** | [h2c-rewriter-traefik](https://github.com/helmfile2compose/h2c-rewriter-traefik) (extension) | POC |

If you use something else (Contour, Ambassador, Istio, AWS ALB, etc.) — standard Ingress `host`/`path`/`backend` fields are always read, so basic routing works, but controller-specific annotations won't translate. You'll either need to [write a rewriter](../developer/extensions/writing-rewriters.md) or configure the reverse proxy manually. Check this *before* investing time in the rest of the setup.

CRDs (Keycloak, cert-manager, trust-manager, Prometheus) are a different story — they're optional, and unsupported CRDs are simply skipped with a warning. Your stack works without them; you just lose the resources they would have produced. Ingress is not optional.

## Installation

Download `helmfile2compose.py` from the [latest h2c-core release](https://github.com/helmfile2compose/helmfile2compose/releases/latest).

If your stack uses CRDs that have an [h2c extension](../catalogue.md) (Keycloak, cert-manager, trust-manager), grab the extension `.py` files from their repos too and drop them in an `extensions/` directory next to the script:

```
extensions/
├── keycloak.py            # from h2c-provider-keycloak
└── cert_manager.py        # from h2c-converter-cert-manager
```

That's it. No package manager needed at this stage — [h2c-manager](h2c-manager.md) is for later, when you ship a `generate-compose.sh` to your users (see [Recommended workflow](#recommended-workflow)).

## Preparing your helmfile

Create a dedicated environment (e.g. `compose`) that disables K8s-only infrastructure. These components have no meaning in compose and will be auto-excluded on first run anyway, but disabling them avoids rendering useless manifests:

```yaml
# environments/compose.yaml
certManager:
  enabled: false
ingress:
  enabled: false
reflector:
  enabled: false
```

Everything else stays enabled — the tool needs to see your Deployments, Services, ConfigMaps, Secrets, and Ingress resources to do its job.

If your stack uses CRDs that have an [h2c extension](../catalogue.md) (Keycloak, cert-manager, trust-manager), keep those enabled — the extensions you should have previously acquired will handle them.

## First run

```bash
python3 helmfile2compose.py --helmfile-dir ~/my-project -e compose \
  --extensions-dir extensions --output-dir .
docker compose up -d
```

This renders the helmfile and converts in one step. Despite the name, **helmfile is not required** — `--from-dir` accepts any directory of pre-rendered K8s YAML (from `helm template`, `kustomize build`, etc.).

On first run, the tool creates `helmfile2compose.yaml` with sensible defaults:
- All PVCs registered as host-path bind mounts under `./data/`
- K8s-only workloads auto-excluded (any workload whose name contains `cert-manager`, `ingress`, or `reflector` — this targets controller Deployments, not Ingress resources)
- Project name derived from the source directory

**Stop here and review `helmfile2compose.yaml`.** You will almost certainly need to:
- Adjust volume paths
- Exclude workloads that make no sense outside K8s (operators, CRD controllers, etc.)
- Add overrides for images that need replacing (e.g. bitnami -> vanilla)

See [Configuration](configuration.md) for the full reference.

### CLI flags

| Flag | Description |
|------|-------------|
| `--helmfile-dir` | Directory containing `helmfile.yaml` or `helmfile.yaml.gotmpl` (default: `.`) |
| `-e`, `--environment` | Helmfile environment (e.g. `compose`, `local`) |
| `--from-dir` | Skip helmfile, read pre-rendered YAML from this directory |
| `--output-dir` | Where to write output files (default: `.`) |
| `--compose-file` | Name of the generated compose file (default: `compose.yml`) |
| `--extensions-dir` | Directory containing extension `.py` files |

`--from-dir` and `--helmfile-dir` serve the same purpose (providing input manifests). If both are specified, `--from-dir` takes priority.

### Output files

- `compose.yml` — services (incl. Caddy reverse proxy), volumes
- `Caddyfile` (or `Caddyfile-<project>` when `disableCaddy: true`) — reverse proxy config from Ingress manifests
- `helmfile2compose.yaml` — persistent config (see [Configuration](configuration.md))
- `configmaps/` — generated files from ConfigMap volume mounts
- `secrets/` — generated files from Secret volume mounts

## Check known issues and workarounds

Before debugging something for hours, check the [known workarounds](known-workarounds/index.md). Recurring tentacles have been identified, sliced, and served — no need to catch the same kraken twice. 

If your stack includes Bitnami charts (Redis, PostgreSQL, Keycloak), install the [bitnami transform](https://github.com/helmfile2compose/h2c-transform-bitnami) — it handles the worst of it automatically.

## What works well

- **Deployments, StatefulSets, DaemonSets, Jobs** — converted to compose services with the right image, env, command, volumes, and ports. Init containers and sidecars get their own services.
- **ConfigMaps and Secrets** — resolved inline into environment variables, or generated as files when volume-mounted.
- **Services** — network aliases (K8s FQDNs resolve natively via compose DNS), alias resolution, port remapping. If your K8s Service remaps port 80 to targetPort 8080, the tool rewrites URLs and env vars automatically.
- **Ingress** — converted to a Caddy reverse proxy with automatic TLS. Path-based routing, host-based routing, catch-all backends. Backend SSL annotations supported.
- **PVCs** — registered in config as bind mounts. `volumeClaimTemplates` (StatefulSets) included.
- **CRDs** — with [extensions](../catalogue.md), Keycloak, cert-manager, and trust-manager CRDs are fully converted.

## What needs manual help

- **Unsupported CRDs** — Zalando PostgreSQL, Strimzi Kafka, etc. are skipped with a warning unless you write an extension. You'll need to add equivalent services manually via the `services:` section in config.
- **Bitnami images** — often need replacing with vanilla equivalents via `overrides:`. Bitnami images have opinions about environment variables, init scripts, and volume paths that don't always translate well.
- **S3 virtual-hosted style** — compose DNS can't resolve `bucket.minio:9000`. Force path-style in your app config and use a `replacement` if needed.
- **CronJobs** — not converted. Run them externally or use a sleep-loop wrapper (but please don't).

See [Limitations](../limitations.md) for the complete list of what gets lost in translation.

## Ingress controllers — details {#ingress-controllers}

Compatibility was covered [above](#before-you-start-ingress-controller). This section documents annotation coverage and configuration.

### Annotations handled

| Controller | Annotations |
|------------|-------------|
| **HAProxy** | `haproxy.org/path-rewrite` → strip prefix, `haproxy.org/server-ssl` + `server-ca` → backend TLS, `server-sni` |
| **Nginx** | `rewrite-target`, `backend-protocol`, `enable-cors`, `proxy-body-size`, `configuration-snippet` (partial) |
| **Traefik** | `router.tls`, standard Ingress path rules. No middleware CRD support. |

HAProxy is built into h2c-core. Nginx and Traefik are extensions — install them with [h2c-manager](h2c-manager.md) or drop the `.py` file in your `extensions/` directory.

### Custom ingress class names

If your cluster uses custom `ingressClassName` values (e.g. `haproxy-internal`, `nginx-dmz`), add a mapping in `helmfile2compose.yaml`:

```yaml
ingressTypes:
  haproxy-internal: haproxy
  haproxy-external: haproxy
  nginx-dmz: nginx
```

Without this, h2c won't recognize the class and the Ingress is skipped with a warning.

## Recommended workflow

1. **One helmfile, two environments.** Keep your K8s environment as-is. Add a `compose` environment that disables cluster-only components. Same charts, same values (mostly), different targets.

2. **Ship a `generate-compose.sh`.** A wrapper script that downloads h2c-manager, installs h2c + extensions, runs the conversion, and maybe generates secrets. See stoatchat-platform or lasuite-platform for examples.

3. **Ship a `helmfile2compose.yaml.template`.** Pre-configure excludes, overrides, and volume mappings that are specific to your project. The generate script copies it to `helmfile2compose.yaml` on first run. Users then customize their copy.

4. **Pin a release.** Use `--core-version` in h2c-manager and `==version` for extensions. Don't point at `main`. The tool's behavior may change between releases (or mutate on its own, I don't know anything at this point). I don't do it myself — but I think we already established my sanity left around v1.2, way before extensions were even on the roadmap.

## The two projects that caused this to exist

- **[stoatchat-platform](https://github.com/baptisterajaut/stoatchat-platform)** — 15 services. The first patient. Worked on the first try, which was the most dangerous outcome.
- **[lasuite-platform](https://github.com/baptisterajaut/lasuite-platform)** — 22 services + 11 init jobs. The second patient. Tentacles started appearing around the volumeClaimTemplates.

Both ship with `generate-compose.sh` and `helmfile2compose.yaml.template`. Reading their setup is probably more useful than anything I could write here.

## Garbage in, garbage out

h2c does **zero validation** of your helmfile output. If your manifests reference a ConfigMap that doesn't exist, a Secret with a missing key, or a Service pointing at a non-existent Deployment — h2c will crash with an ugly Python traceback, not a helpful error message.

This is by design. Error handling for malformed K8s manifests is not h2c's job — it would massively increase complexity for something that `helmfile lint`, `helm template --validate`, and `kubectl apply --dry-run` already do. h2c assumes its input is valid. If it isn't, the consequences are yours.

**Make sure your helmfile works on a real Kubernetes cluster first.** A real runtime — one with an actual apiserver, actual controllers, actual sanity — should have validated the output before h2c ever sees it. h2c is a downstream consumer, not a linter. Fix your helmfile, re-render, re-convert. Actions, consequences.

## Final warning

This tool works. It has been tested on real helmfiles with real users. But it is, fundamentally, an act of desecration — stripping Kubernetes of everything that makes it Kubernetes (scheduling, scaling, self-healing, network policies, RBAC) and leaving behind a flat list of containers. Every edge case you hit is a reminder that you are running something that was designed for orchestration on a machine that has no orchestra — and that the person who asked you for a docker-compose owes you a drink — or the psychiatric bill, whichever comes first.

> *The temple was not translated — it was dismantled, stone by stone, and rebuilt as a shed. The prayers still worked. The architect watched, powerless, as the faithful praised the shed.*
>
> — *De Vermis Mysteriis, On Unnecessary Simplifications (debatable)*
