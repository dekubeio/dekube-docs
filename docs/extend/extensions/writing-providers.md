# Writing providers

A provider emulates a Kubernetes controller — without the inconvenience of actually running one. It reads CRDs, pretends a reconciliation loop happened, and produces the compose services that the real controller would have created. Same manifests in, same services out, minus the control plane, the RBAC, and the existential purpose. Providers subclass `Provider` (from `dekube`) and share the same `kinds` + `convert()` interface as converters, but deal with additional patterns: injecting synthetic resources, registering services, and coordinating with other extensions through priority ordering.

> *The priests were gone, yet the rituals continued. The faithful did not notice — for the incantations were spoken in the same tongue, and the sacrifices consumed in the same fire. Only the altar knew that the hands were different.*
>
> — *De Vermis Mysteriis, On the Succession of Clergy (contested)*

Read [Writing converters](writing-converters.md) first for the base interface (`kinds`, `convert()`, `ConvertResult`, `ConvertContext`, priority, testing, publishing).

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
    priority = 10
```

The distinction is enforced — `Provider` is a base class in `dekube.pacts.types` (subclass of `Converter`, default priority 500). Subclassing it signals intent to the framework. Naming convention: `dekube-provider-*` for providers, `dekube-converter-*` for converters.

## Injecting synthetic resources

This is where the forgery happens. Converters can fabricate K8s resources that never existed in any cluster — Secrets the operator would have generated, ConfigMaps the controller would have reconciled — and inject them into `ctx` as if they'd always been there. Downstream converters and the volume-mount machinery consume them without question.

### Secrets

```python
# Inject a synthetic Secret (cert-manager pattern)
ctx.secrets["my-tls-secret"] = {
    "metadata": {"name": "my-tls-secret"},
    "stringData": {
        "tls.crt": pem_cert,
        "tls.key": pem_key,
    },
}

# Write files to disk so volume mounts can find them
import os
secret_dir = os.path.join(ctx.output_dir, "secrets", "my-tls-secret")
os.makedirs(secret_dir, exist_ok=True)
with open(os.path.join(secret_dir, "tls.crt"), "w", encoding="utf-8") as f:
    f.write(pem_cert)
ctx.generated_secrets.add("my-tls-secret")
```

Use `stringData` (not `data`) to avoid double-encoding — the main pipeline handles base64 decoding for `data` entries, but synthetic secrets should use plain text.

### ConfigMaps

```python
# Inject a synthetic ConfigMap (trust-manager pattern)
ctx.configmaps["my-ca-bundle"] = {
    "metadata": {"name": "my-ca-bundle"},
    "data": {"ca-certificates.crt": pem_bundle},
}

# Write files to disk so volume mounts can find them
import os
cm_dir = os.path.join(ctx.output_dir, "configmaps", "my-ca-bundle")
os.makedirs(cm_dir, exist_ok=True)
with open(os.path.join(cm_dir, "ca-certificates.crt"), "w", encoding="utf-8") as f:
    f.write(pem_bundle)
ctx.generated_cms.add("my-ca-bundle")
```

Same principle as Secrets — inject into `ctx.configmaps` so downstream converters can read the data, and write to disk so the volume-mount machinery picks it up.

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
    priority = 10  # generates secrets

class TrustManagerConverter:
    kinds = ["Bundle"]
    priority = 20  # reads cert-manager's secrets
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
