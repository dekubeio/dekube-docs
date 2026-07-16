---
description: "Writing a dekube provider: emulating a Kubernetes controller to produce compose services, inject synthetic resources, and register aliases."
---

# Writing providers

A provider emulates a Kubernetes controller — without the inconvenience of actually running one. It reads CRDs, pretends a reconciliation loop happened, and produces the compose services that the real controller would have created. Same manifests in, same services out, minus the control plane, the RBAC, and the existential purpose. Providers subclass `Provider` (from `dekube`) and share the same `kinds` + `convert()` interface as converters, but deal with additional patterns: injecting synthetic resources, registering services, and coordinating with other extensions through priority ordering.

> *The priests were gone, yet the rituals continued. The faithful did not notice — for the incantations were spoken in the same tongue, and the sacrifices consumed in the same fire. Only the altar knew that the hands were different.*
>
> — *De Vermis Mysteriis, On the Succession of Clergy (contested)*

Read [Writing converters](writing-converters.md) first for the base interface (`kinds`, `convert()`, `ConverterResult`, `ConvertContext`, priority, testing, publishing).

## Provider vs Converter

A converter tells the engine what a CRD *means* — it reads manifests and injects synthetic resources for others to consume. A provider tells the engine what a CRD *does* — it pretends a controller ran and produces the compose services that controller would have created. The converter forges documents; the provider forges the entire bureaucracy.

CRD extensions that **produce compose services** (i.e. return non-empty `ProviderResult.services`) should subclass `Provider`:

```python
from dekube import Provider, ProviderResult

class KeycloakProvider(Provider):
    kinds = ["Keycloak", "KeycloakRealmImport"]
    name = "keycloak"
    # Provider default priority is 500 — override if needed
```

CRD extensions that only **inject synthetic resources** (Secrets, ConfigMaps) without producing services should remain plain converters:

```python
from dekube import ConverterResult

class CertManagerConverter:
    kinds = ["Certificate", "ClusterIssuer", "Issuer"]
    priority = 100
```

The distinction is enforced — `Provider` is a base class in `dekube.pacts.types` (subclass of `Converter`, default priority 500). Subclassing it signals intent to the framework. Naming convention: `dekube-provider-*` for providers, `dekube-converter-*` for converters.

## Injecting synthetic resources

This is where the forgery happens. Converters can fabricate K8s resources that never existed in any cluster — Secrets the operator would have generated, ConfigMaps the controller would have reconciled — and inject them into `ctx` as if they'd always been there. Downstream converters and the volume-mount machinery consume them without question.

### Secrets

```python
from dekube import write_secret_files

# Inject a synthetic Secret (cert-manager pattern)
ctx.secrets["my-tls-secret"] = {
    "metadata": {"name": "my-tls-secret"},
    "stringData": {
        "tls.crt": pem_cert,
        "tls.key": pem_key,
    },
}

# Emit its data to disk so volume mounts can find it. The helper creates
# output_dir/secrets/my-tls-secret/, writes each key as a file, and records
# the name in ctx.generated_secrets. Returns "./secrets/my-tls-secret".
write_secret_files("my-tls-secret", ctx)
```

Use `stringData` (not `data`) to avoid double-encoding — the main pipeline handles base64 decoding for `data` entries, but synthetic secrets should use plain text. `write_secret_files` handles both (plus `binaryData`) and warns rather than crashing on a path-traversal key. Don't hand-roll `os.makedirs` + `open` + `ctx.generated_secrets.add` — that's what the helper is for.

### ConfigMaps

```python
from dekube import write_configmap_files

# Inject a synthetic ConfigMap (trust-manager pattern)
ctx.configmaps["my-ca-bundle"] = {
    "metadata": {"name": "my-ca-bundle"},
    "data": {"ca-certificates.crt": pem_bundle},
}

# Emit its data to output_dir/configmaps/my-ca-bundle/ and record it in
# ctx.generated_cms. Returns "./configmaps/my-ca-bundle".
write_configmap_files("my-ca-bundle", ctx)
```

Same principle as Secrets — inject into `ctx.configmaps` so downstream converters can read the data, then call `write_configmap_files` so the volume-mount machinery finds it on disk.

### Generated credentials

Operators often auto-generate a password and store it in a Secret. Emulate that with `generate_password` (random alphanumeric, default 24 chars — pass the operator's length if it differs), and read a value back from any Secret in `ctx.secrets` with `secret_value`:

```python
from dekube import generate_password, secret_value, write_secret_files

password = generate_password(32)
ctx.secrets["db-credentials"] = {
    "metadata": {"name": "db-credentials"},
    "stringData": {"password": password},
}
write_secret_files("db-credentials", ctx)

# Later, or in a downstream converter reading a Secret someone else injected:
password = secret_value(ctx.secrets["db-credentials"], "password")
```

The cnpg and keycloak providers use exactly this pattern — generate the credential the operator would have created, write it to disk, and hand the same value to every service that references the Secret.

## Registering network aliases

If your converter creates services that don't exist in the rendered manifests (e.g. Keycloak — the K8s operator creates the Service at runtime), register them in `ctx.services_by_selector` and `ctx.alias_map` so the network alias builder generates FQDN aliases:

```python
# Register the compose service
ctx.services_by_selector[name] = {
    "name": name,
    "namespace": namespace,
    "selector": {...},
    "type": "ClusterIP",
    "ports": [...],
}

# If the K8s Service name differs from the compose service name,
# register both in services_by_selector + an alias_map entry
k8s_svc_name = f"{name}-service"
ctx.alias_map[k8s_svc_name] = name
ctx.services_by_selector[k8s_svc_name] = {
    "name": k8s_svc_name,
    "namespace": namespace,
    "selector": {...},
    "type": "ClusterIP",
    "ports": [...],
}
```

The `namespace` field is required for FQDN alias generation. Without it, only short-name aliases are created.

## Cross-converter dependencies

Use `priority` to ensure converters run in the right order when they depend on each other's output:

```python
class CertManagerConverter:
    kinds = ["Certificate", "ClusterIssuer", "Issuer"]
    priority = 100  # generates secrets

class TrustManagerConverter:
    kinds = ["Bundle"]
    priority = 200  # reads cert-manager's secrets
```

The cert-manager converter injects synthetic Secrets into `ctx.secrets`. The trust-manager converter reads those secrets to assemble CA bundles. Without priority ordering, trust-manager might run first and find nothing.

## The emulation boundary

A CRD converter doesn't *run* the K8s controller — it *replaces* it. Where do you stop pretending? That's the question every provider author faces, and the answer is always "one step further than you planned."

The controller watches CRDs, reconciles state, creates/updates resources. The converter reads the same CRDs once, at conversion time, and produces the final output directly. No watch. No loop. No "eventually consistent" — just "consistent, once, right now."

This means:

- **No reconciliation loop** — the converter runs once. If something changes, you re-run dekube. The controller's patient vigil becomes a one-shot invocation.
- **No runtime state** — the converter can't watch for changes or react to failures. What it produces is final.
- **No side effects** — the converter shouldn't create containers or processes. It produces compose service definitions (providers) or synthetic resources (converters), or both.

The goal is to produce the same *result* the controller would have produced, without the *process*. For most CRDs, this is straightforward — the CR is a declaration, and the controller's job is to materialize it. The converter does the same materialization, at a different time. For the CRDs where this *isn't* straightforward — where the controller has opinions, state machines, or runtime decisions — see [the wall](../../understand/concepts.md#the-emulation-boundary) and decide how close you're willing to stand.
