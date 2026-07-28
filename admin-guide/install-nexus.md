# Install Muxon Nexus (Enterprise)

**Muxon Nexus** is an enterprise add-on layered on top of **Muxon Core**. You should complete [Install Muxon Core](install-core.md) (or install Core as part of the same Helm release) before relying on Nexus-specific features.

This document assumes you have access to the **`muxon-nexus`** Helm chart and container images (enterprise build).

---

## Prerequisites

- **Muxon Core** running and healthy (database migrated, OIDC working).
- **Enterprise license** and images for Nexus services (API gateway, director, etc.) as provided by your vendor.
- Same baseline requirements as Core for your chosen platform: **Docker** or **Kubernetes + Helm**, or the **enterprise appliance** image.
- **Helm 3.14+** if using Kubernetes (parent chart bundles `muxon-core` as a dependency).

---

## What Nexus adds

The `muxon-nexus` Helm chart typically:

- Deploys enterprise microservices alongside **Muxon Core**.
- Pulls **muxon-core** in as a **subchart** (`muxon-core` values are nested under the `muxon-core:` key).

You still pass **secrets** and **initial configuration** for Core using the same model as open-source Core, but with **nested values**.

## Bootstrap in Enterprise (`nexus-initializer`)

Enterprise deployments use a separate init step called **`nexus-initializer`**.

- It runs the **OSS bootstrap** first (Flyway OSS migrations + initial seed + per-service config files)
- Then it runs **enterprise migrations** and writes `nexus-services-application.yaml`

In other words, **enterprise can run only `nexus-initializer`** as the init container; you do not need to run `muxon-initializer` separately.

---

## Option A — Docker Compose

There is no single standard `docker-compose` file for Nexus in this repository; enterprise deployments usually ship a **vendor compose override** or additional services.

**Recommended approach:**

1. Install or run **Muxon Core** using [compose/muxon.yml](../../compose/muxon.yml) (or your hardened variant).
2. Add Nexus services from your enterprise delivery:
   - Either merge a second compose file:  
     `docker compose -f compose/muxon.yml -f nexus.override.yml up -d`
   - Or define Nexus services in the same project with `depends_on: core-services`.

3. Reuse the **same** secret files for Core (`muxon-passphrase`, `muxon-oidc-secret`, `muxon-db-password`) where applicable; Nexus components may need **extra** secrets — follow the enterprise README shipped with your images.

---

## Option B — Kubernetes / Helm (`muxon-nexus` chart)

### Step 1 — Package the chart (if installing from source)

From the **`muxon-build`** repo (or your CI):

```bash
VARIANT=enterprise ./scripts/package-helm.sh
```

This produces `dist/helm/muxon-nexus-<version>.tgz` with the `muxon-core` dependency vendored under `charts/`.

### Step 2 — Create secrets (same as Core)

```bash
kubectl create namespace muxon
kubectl create secret generic muxon-secrets \
  --namespace muxon \
  --from-file=muxon-passphrase=./secrets/muxon-passphrase \
  --from-file=muxon-oidc-secret=./secrets/muxon-oidc-secret \
  --from-file=muxon-db-password=./secrets/muxon-db-password \
  --from-file=keycloak-admin-password=./secrets/keycloak-admin-password
```

### Step 3 — Values file

Create `nexus-values.yaml`. **Nest** Core settings under `muxon-core:`.

**Bundled Postgres + Keycloak example:**

```yaml
muxon-core:
  postgres:
    enabled: true
  keycloak:
    enabled: true
    adminExistingSecret: muxon-secrets
    adminSecretKey: keycloak-admin-password
  bootstrap:
    existingSecret: muxon-secrets
  image:
    repository: your-registry.example.com/core-services
    tag: "0.1.0"

enterprise:
  enabled: true
```

**External database / IdP:** set `muxon-core.postgres.enabled: false`, `muxon-core.keycloak.enabled: false`, and set `muxon-core.bootstrap.initialConfig` URLs accordingly (or use `--set-file` as below).

### Step 4 — Install with `initial-config.yaml`

Pass Core’s bootstrap file into the **subchart** using a **double** prefix on `--set-file`:

```bash
helm upgrade --install muxon-enterprise ./muxon-nexus-0.1.0.tgz \
  --namespace muxon \
  -f nexus-values.yaml \
  --set-file muxon-core.bootstrap.initialConfigYaml=./initial-config.yaml
```

### Step 5 — Verify

```bash
kubectl -n muxon get pods
kubectl -n muxon logs deploy/core-services -c core-services --tail=50
```

Inspect Nexus-specific deployments from your chart (`kubectl get deploy -n muxon`).

---

## Option C — VM appliance (enterprise variant)

Build the appliance with the **enterprise** variant so the image contains Nexus charts and images (see **`muxon-build`** `Makefile` / `README.md`, target such as `appliance-enterprise`).

On first boot, **`muxon-firstboot.sh`** selects **`muxon-nexus-*.tgz`** if present under `/opt/muxon/charts/` and installs it with generated values that enable bundled Postgres and Keycloak for **`muxon-core`**.

Operator **`initial-config.yaml`** is passed with:

```bash
--set-file muxon-core.bootstrap.initialConfigYaml=/etc/muxon/initial-config.yaml
```

You can still override Helm values by placing **`/etc/muxon/helm-values.overrides.yaml`** on the VM before first boot (merged by the first-boot script).

---

## Verifying Nexus features

- Confirm enterprise **feature flags** or config maps created by the chart (name varies by release).
- Use your vendor’s health checks or admin UI entry points.
- Ensure **Core** `/actuator/health` remains **UP** behind Nexus routing if an API gateway is used.

---

## Troubleshooting

| Issue | What to check |
|-------|----------------|
| Subchart values ignored | Keys must be under `muxon-core:` in the parent values file. |
| `set-file` has no effect | Use `muxon-core.bootstrap.initialConfigYaml=...`, not `bootstrap.initialConfigYaml` alone. |
| Image pull errors | `muxon-core.image.repository` / `tag` must match your registry after `package-helm.sh` rewrites. |
| Core works, Nexus pods crash | Enterprise image names, pull secrets, and license volumes — see enterprise documentation. |

For Core-only installation and cloud-init examples, see [Install Muxon Core](install-core.md).
