# CRD patterns

This page covers patterns specific to converters that handle CRD kinds — emulating what a K8s controller would have done at runtime. The code interface is the same (`kinds` + `convert()` → `ConvertResult`), but CRD converters deal with additional patterns: injecting synthetic resources, registering services that the K8s controller would have created, and coordinating with other converters through priority ordering.

> *The priests were gone, yet the rituals continued. The faithful did not notice — for the incantations were spoken in the same tongue, and the sacrifices consumed in the same fire. Only the altar knew that the hands were different.*
>
> — *De Vermis Mysteriis, On the Succession of Clergy (contested)*

Read [Writing converters](writing-converters.md) first for the base interface (`kinds`, `convert()`, `ConvertResult`, `ConvertContext`, priority, testing, publishing).

## Injecting synthetic resources

Converters can inject synthetic Secrets or ConfigMaps into `ctx` for other converters or the existing volume-mount machinery to consume:

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

A CRD converter doesn't *run* the K8s controller — it *replaces* it. The controller watches CRDs, reconciles state, creates/updates resources. The converter reads the same CRDs once, at conversion time, and produces the final output directly.

This means:

- **No reconciliation loop** — the converter runs once. If something changes, you re-run h2c.
- **No runtime state** — the converter can't watch for changes or react to failures.
- **No side effects** — the converter shouldn't create containers or processes. It produces compose service definitions (providers) or synthetic resources (converters), or both.

The goal is to produce the same *result* the controller would have produced, without the *process*. For most CRDs, this is straightforward — the CR is a declaration, and the controller's job is to materialize it. The converter does the same materialization, at a different time.
