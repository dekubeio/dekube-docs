# Glossary

Quick reference for terms used across this documentation. If a word sounds like it belongs in a Lovecraftian grimoire, it probably does — but it also has a technical meaning.

---

**bind mount**
:   A Docker volume type that maps a host directory directly into the container (`./data/postgres:/var/lib/postgresql/data`). Changes are visible on both sides. Contrast with *named volumes*, which Docker manages internally. helmfile2compose converts K8s PVCs to bind mounts by default, so data lives on disk in predictable paths.

**ConfigMap**
:   A Kubernetes object that stores non-sensitive configuration data as key-value pairs. helmfile2compose resolves ConfigMap references inline into environment variables and writes ConfigMap files to disk for volume mounts.

**converter**
:   An extension that handles one or more K8s resource `kinds` and produces *synthetic resources* (Secrets, ConfigMaps, files) without creating compose services. Example: the cert-manager converter reads `Certificate` CRDs and generates PEM files. See also: *provider*, *indexer*.

**dispatch**
:   The process of routing each K8s manifest to the converter (or provider, or indexer) that handles its `kind`. Converters are sorted by *priority* (lower = earlier) and each manifest is handed to the first converter that claims its kind. "Dispatch loop" = the core engine iterating over all manifests and calling the appropriate `convert()` method.

**distribution**
:   A packaged combination of *h2c-core* + a set of bundled extensions, concatenated into a single `.py` file. The default distribution is `helmfile2compose` (core + the *Eight Monks*). `kubernetes2simple` is a stacked distribution (helmfile2compose + all official extensions). Think of it like a K8s distribution (k3s, EKS) vs the bare apiserver.

**emulation boundary**
:   The line between what h2c can convert and what it cannot. *Tier 1* (flattened): K8s declarations materialized as compose equivalents. *Tier 2* (ignored): operational K8s features that don't affect app behavior (RBAC, HPA, NetworkPolicies). *Tier 3* (the wall): anything that requires a live kube-apiserver at runtime — now breachable via the [fake-apiserver](catalogue.md#fake-apiserver) extension, which we do not endorse, recommend, or acknowledge. See [Concepts](developer/concepts.md#the-emulation-boundary).

**extension**
:   A single-file Python module (`.py`) that plugs into h2c-core's contracts. Can be a *converter*, *provider*, *indexer*, *transform*, *rewriter*, or *ingress provider*. Installed via `--extensions-dir` or through *h2c-manager*. Each extension is also a GitHub repo for distribution purposes.

**FQDN** (Fully Qualified Domain Name)
:   The complete DNS name of a K8s service: `service.namespace.svc.cluster.local`. helmfile2compose preserves these as Docker Compose network aliases so that certificates, Prometheus targets, and hardcoded URLs continue to work without rewriting.

**h2c-core**
:   The bare conversion engine — pipeline, extension loader, CLI, empty registries. Converts nothing on its own. Users never interact with it directly; they use a *distribution*. Extension developers and distribution builders work with h2c-core. Produces `h2c.py`.

**helmfile2compose**
:   Two things depending on context: (1) the GitHub org and overall project name, (2) the default *distribution* — h2c-core + the *Eight Monks*, assembled into `helmfile2compose.py`. When the docs say "helmfile2compose" without qualifier, they usually mean the distribution.

**indexer** (IndexerConverter)
:   A converter subtype that populates `ConvertContext` lookups (e.g. `ctx.configmaps`, `ctx.secrets`, `ctx.services`) without producing any output. Indexers run first (default priority 50) so that later converters and providers can resolve references. The four bundled indexers handle ConfigMap, Secret, PVC, and Service.

**ingress entry**
:   A dict produced by an *ingress rewriter* describing one routing rule: hostname, path, upstream, TLS settings, extra directives. The *ingress provider* consumes these entries to build the reverse proxy config (e.g. a Caddyfile).

**ingress provider** (IngressProvider)
:   The extension that produces the reverse proxy service and its config file from *ingress entries*. The default is CaddyProvider (generates a Caddy service + Caddyfile). Replacing it swaps the entire reverse proxy backend. Not to be confused with *ingress rewriter*.

**ingress rewriter** (IngressRewriter)
:   An extension that translates controller-specific Ingress annotations (HAProxy, nginx, traefik) into *ingress entries*. Each rewriter targets one ingress controller via `ingressClassName`. Not to be confused with *ingress provider* — the rewriter reads annotations, the provider builds the proxy.

**named volume**
:   A Docker-managed volume (`postgres-data:/var/lib/postgresql/data`). Docker handles the storage location. Contrast with *bind mount*.

**network alias**
:   A DNS name that Docker Compose registers for a service on its network. helmfile2compose adds K8s FQDN variants as aliases so that services can reach each other using the same DNS names they used in Kubernetes.

**pacts**
:   The stable public API that extensions import from h2c-core (`from h2c import ...`). Located in `src/h2c/pacts/`. Includes `ConvertContext`, `ConverterResult`, `ProviderResult`, base classes, and helper functions. Guaranteed stable across minor versions. The Lovecraftian name reflects the project's tone — "pacts" = "contracts."

**provider**
:   A converter subtype that produces compose services (and possibly synthetic resources). Subclasses `Provider` (default priority 500). Example: the Keycloak provider reads `Keycloak` CRDs and produces a compose service with the right env vars. See also: *converter*, *indexer*.

**PVC** (PersistentVolumeClaim)
:   A Kubernetes object that requests storage. helmfile2compose converts PVCs to bind mounts, mapping each claim to a host directory configured in `helmfile2compose.yaml`.

**Secret**
:   A Kubernetes object that stores sensitive data (passwords, tokens, certificates). helmfile2compose resolves Secret references into plain-text environment variables or files. Yes, the security implications are exactly what you think.

**synthetic resource**
:   A K8s resource (Secret, ConfigMap) that doesn't come from the original manifests but is *created by a converter* during conversion. Example: the cert-manager converter generates Certificate Secrets that didn't exist in the input manifests. Other converters and providers can then reference these synthetic resources as if they were real.

**The Eight Monks**
:   The eight extensions bundled in the helmfile2compose distribution. Four indexers (Librarian, Guardian, Binder, Weaver), one workload provider (Builder), one ingress rewriter (Herald), one ingress provider (Gatekeeper), and one transform (Custodian). Together they convert standard K8s manifests — everything beyond standard resources requires additional extensions.

**transform**
:   An extension that post-processes the final compose output *after* all converters have run. Transforms see the complete `compose_services` dict and `ingress_entries` list and can mutate them in place. Example: `flatten-internal-urls` strips network aliases and rewrites FQDNs. Transforms have no `kinds` — they operate on the output, not the input.
