# Testing

It has tests. It has an executioner.

A project that emulates a container orchestrator by flattening its output into a different container orchestrator, whose primary architectural document is a fake Lovecraftian grimoire, whose fake kube-apiserver lives on a personal GitHub account for plausible deniability — this project now has automated regression testing. With CI. And a torturer.

> *And lo, the disciples returned to the temple carrying instruments of verification — not to prove the rituals false, but to ensure they failed in precisely the same way, every time, without exception. The high priest wept, for he understood: reproducible heresy is still heresy, but it is heresy you can ship.*
>
> — *Necronomicon, On the Formalization of Abomination (apparently)*

## What it tests

[dekube-testsuite](https://github.com/dekubeio/dekube-testsuite) compares dekube output between a **pinned reference version** and the **latest release**. The test harness uses dekube-manager (rolling from `main`) as the runner — it's the tool, not the subject. What's compared is core + extension output between versions.

### Regression

The test runner (`run-tests.sh`) downloads both versions, generates a `dekube.yaml` for each, and diffs the output. It runs multiple extension combos:

1. **Core only** — no extensions, baseline behavior
2. **Each extension individually** — isolation testing
3. **All extensions together** — interaction testing

A diff is not a failure. A diff is information. When you bump core from v3.0.1 to v3.1.0 and the caddy service disappears from the output, that's the executioner doing its job — it tells you the refactoring changed behavior, and you decide whether that's intentional.

### Static manifests

The `manifests/` directory contains edge cases organized by kind:

| File | What it covers |
|------|---------------|
| `deployments.yaml` | Single-container, multi-container, probes, command/args, securityContext, hostNetwork, serviceAccount |
| `statefulsets.yaml` | volumeClaimTemplates, headless services, updateStrategy, init containers |
| `jobs.yaml` | Basic jobs, init containers, sidecars, restartPolicy variations |
| `services.yaml` | ClusterIP, multi-port, ExternalName, ExternalName chain, NodePort, LoadBalancer |
| `ingress.yaml` | Single/multi-path, TLS, HAProxy annotations (server-ssl, path-rewrite, server-ca) |
| `configmaps-secrets.yaml` | envFrom, volume mounts, binary data, shared references across deployments |
| `crds.yaml` | KeycloakRealmImport, Certificate, ClusterIssuer, Issuer, ServiceMonitor, Bundle |
| `edge-cases.yaml` | Empty docs, 63/64-char names, missing namespace, no selector, empty containers, unknown kinds |

### The torturer (`--perf N`) {#the-torturer}

The generator (`generate.py`) produces manifests at O(n³) scale:

- `n` releases, each with `n` Deployments, `n` ConfigMaps, `n` Secrets, `n` Services, 1 Ingress
- Each Deployment mounts all `n` ConfigMaps of its release
- Total volume resolutions: n² deployments × n mounts = **n³**

| n | Deployments | ConfigMap mounts | Approx. time |
|---|-------------|------------------|--------------|
| 5 | 25 | 125 | < 1s |
| 15 | 225 | 3,375 | seconds |
| 30 | 900 | 27,000 | notable |
| 50 | 2,500 | 125,000 | pain |

Performance tests run locally, not in CI — public runners aren't meant for this.

## Usage

```bash
# Full regression
./run-tests.sh

# Override reference core version
./run-tests.sh --core v2.0.0

# Override reference extension version
./run-tests.sh --ext keycloak==v0.1.0

# Torture test
./run-tests.sh --perf 15

# Keep /tmp output for inspection
./run-tests.sh --perf 15 --keep
```

## Reference versions

Edit `dekube-known-versions.json` to bump the pinned reference:

```json
{
  "reference": {
    "core": "v3.0.1",
    "extensions": {
      "cert-manager": "v0.2.0",
      "keycloak": "v0.3.0",
      "servicemonitor": "v0.2.0",
      "trust-manager": "v0.2.0",
      "nginx": "v0.2.0",
      "traefik": "v0.2.0",
      "flatten-internal-urls": "v0.2.0",
      "bitnami": "v0.2.0"
    },
    "exclude-ext-all": ["flatten-internal-urls"]
  }
}
```

Extensions listed here are tested individually; unlisted are skipped. Extensions in `exclude-ext-all` are tested in isolation but excluded from the combined `ext-all` combo (e.g. due to incompatibilities declared in the registry).

## CI

The GitHub Actions workflow runs regression weekly (Monday 6am UTC) and on push to test-related files. Diffs are uploaded as artifacts for human review. There are no assertions — the diff *is* the output.

## Reading diffs

- `core-only` diff → pure dekube-engine behavioral change
- `ext-<name>` diff → change in that extension or its interaction with core
- `ext-all` diff → interaction between extensions

When you see a diff, the question isn't "is this a bug?" — it's "did I mean to change this?" If yes, bump the reference version in `dekube-known-versions.json` and the diff disappears. If no, you just caught a regression.
