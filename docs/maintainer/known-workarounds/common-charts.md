# Common charts that don't comply

Some Helm charts — popular, well-maintained, widely deployed — produce manifests that don't survive the flattening intact. Not because h2c is wrong, but because the charts make assumptions about the runtime that only hold in Kubernetes.

> *Certain beasts, though domesticated by the ancients, refuse to kneel before lesser altars. They do not resist — they simply ignore the new prayers, continuing to chant the old liturgy until the disciple learns to speak their tongue.*
>
> — *Unaussprechlichen Kulten, On Stubborn Familiars (more or less)*

---

## MinIO

**Chart:** `minio/minio`

MinIO itself converts fine — the image is standard, the ports are standard, the environment is standard. The problem is the `minio-post-job`: a Kubernetes Job that runs `mc` (the MinIO client) to create buckets after MinIO starts. In K8s, the Job controller retries until MinIO is ready. In compose, a `restart: on-failure` service might work, but the post-job often has cluster-specific configuration, hardcoded bucket names, or policies that don't apply locally.

The cleaner approach: exclude the post-job and write your own init service.

**Config:**

```yaml
exclude:
- minio-post-job

services:
  minio-init:
    image: quay.io/minio/mc:latest
    restart: on-failure
    entrypoint:
    - /bin/sh
    - -c
    command:
    - >-
      mc alias set local http://minio:9000
      $secret:minio:rootUser $secret:minio:rootPassword
      && mc mb --ignore-existing local/my-bucket
      && mc mb --ignore-existing local/my-other-bucket
```

Replace the bucket names with whatever your application expects.

**Why not just let the post-job run?**

You can try. Sometimes it works. But the post-job often carries policies, lifecycle rules, or notification configurations that reference K8s-specific endpoints. And the `mc` image tag pinned in the chart may be stale. A hand-written init service with `mc:latest` and just the bucket creation is more predictable.

**Also:** if your application uses S3 virtual-hosted style URLs (`bucket.minio:9000`), compose DNS can't resolve the dotted hostname. Add a replacement to switch to path-style access:

```yaml
replacements:
- old: "path_style_buckets = false"
  new: "path_style_buckets = true"
```

The exact config key depends on your application (Django: `AWS_S3_ADDRESSING_STYLE: path`, Rails: `force_path_style: true`, etc.).

**Seen in:** lasuite-platform, stoatchat-platform.

---

## Bitnami charts (Redis, PostgreSQL, Keycloak)

Want to fix Bitnami shenanigans? Just install the bitnami transform. We desecrate the temple by deconstructing it — they do it by existing.

```bash
python3 h2c-manager.py bitnami
```

The transform detects Bitnami images, applies the fixes below automatically, and logs every modification to stderr. Manual overrides still take precedence if you need fine-grained control.

Or dive deep into their treachery yourself:

### Bitnami PostgreSQL

**Chart:** `bitnami/postgresql`

The Bitnami PostgreSQL chart wraps the standard `postgres` image in a custom entrypoint that manages permissions, TLS, replication setup, and init scripts through a chain of shell scripts baked into the image at `/opt/bitnami/`. In Kubernetes, this works because the chart controls the full pod spec — volumes, security context, environment — and everything lands exactly where the scripts expect.

In compose, h2c converts the pod faithfully, but the Bitnami entrypoint expects its data directory at `/bitnami/postgresql` (not the standard `/var/lib/postgresql/data`), reads secrets from `/opt/bitnami/postgresql/secrets/`, and runs init scripts from a specific path. The generated volume mounts often don't line up, because h2c maps PVCs to host paths without knowledge of Bitnami's internal conventions.

**Override:**

```yaml
overrides:
  <release>-postgresql:
    volumes:
    - $volume_root/postgresql:/bitnami/postgresql
    - ./secrets/<release>-postgresql:/opt/bitnami/postgresql/secrets:ro
    - ./configmaps/<release>-postgresql-init-scripts:/docker-entrypoint-initdb.d:ro
```

Replace `<release>` with your Helm release name (e.g. `lasuite`, `shared-database`).

**What this does:**

- Mounts the data directory where Bitnami expects it (`/bitnami/postgresql`)
- Mounts the generated secrets (passwords, TLS certs) where the entrypoint reads them
- Mounts init scripts to the standard `docker-entrypoint-initdb.d/` path — these are the SQL scripts that create databases, users, and grants on first startup

**Seen in:** lasuite-platform, my proprietary production helmfile.

### Bitnami Redis

**Chart:** `bitnami/redis`

The Bitnami Redis chart generates a `redis-master` service with a multi-layered entrypoint: a shell script that sets up permissions, writes a dynamic `redis.conf`, configures ACLs, and eventually starts `redis-server`. The entrypoint references paths, environment variables, and volume mounts that are tightly coupled to Bitnami's image layout.

In compose, the entrypoint chain tends to fail silently or hang — it expects specific directory structures and permission schemes that don't exist outside the Bitnami pod spec. The simplest fix is to bypass all of it: replace the image with stock Redis and configure it directly.

**Override:**

```yaml
overrides:
  redis-master:
    image: redis:7-alpine
    entrypoint: null
    command:
    - redis-server
    - --requirepass
    - $secret:redis:redis-password
    volumes:
    - $volume_root/redis:/data
    environment: null
```

**What this does:**

- Replaces the Bitnami image with stock `redis:7-alpine`
- Nulls out the entrypoint and environment (the Bitnami env vars are meaningless without the Bitnami entrypoint)
- Sets a direct `redis-server --requirepass` command, pulling the password from the K8s Secret
- Mounts a simple data volume at `/data` (the standard Redis path)

This is a full replacement, not a tweak. The Bitnami Redis chart's value proposition is operational tooling (sentinel, replication, metrics) — none of which applies in a single-machine compose stack. Stock Redis with a password is all you need.

**Seen in:** lasuite-platform, stoatchat-platform.

### Bitnami Keycloak

**Chart:** `bitnami/keycloak`

The Bitnami Keycloak chart wraps the standard Keycloak image with an init container (`prepare-write-dirs`) that copies configuration to emptyDir volumes, and reads admin/database passwords from mounted Secret files. In compose, the emptyDir copy init fails (no shared emptyDir between services), and the Secret file mounts may not exist.

**Override:**

```yaml
overrides:
  <release>-keycloak:
    environment:
      KC_BOOTSTRAP_ADMIN_PASSWORD: $secret:<release>-keycloak:admin-password
      KC_DB_PASSWORD: $secret:<release>-postgresql:password

exclude:
- <release>-keycloak-init-prepare-write-dirs
```

Replace `<release>` with your Helm release name.

**What this does:**

- Injects the admin and database passwords as environment variables instead of file mounts
- Excludes the `prepare-write-dirs` init container that fails on emptyDir

**Seen in:** mijn-bureau-infra.
