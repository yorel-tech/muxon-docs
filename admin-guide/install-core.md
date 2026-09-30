# Install Muxon Core

This guide explains how to install **Muxon Core** using:

- **Docker Compose** (good for single-host and trials)
- **Kubernetes with Helm** (good for production clusters)
- **VM appliance** (OVA/QCOW2 with embedded k3s)

You can use **bundled** PostgreSQL and Keycloak, or **your own** database and identity provider.

---

## What you will provide

| Item | Purpose |
|------|---------|
| **`initial-config.yaml`** | Instance name, admin emails, IdP URLs, JDBC URL, tenant defaults. **Passwords for the database and OIDC client are not trusted from this file for runtime** — they come from secret files (Docker/Kubernetes) or generated files (appliance). |
| **`muxon-passphrase`** | Used with instance metadata to derive encryption keys for stored secrets. |
| **`muxon-oidc-secret`** | OIDC client secret (must match your IdP client configuration). |
| **`muxon-db-password`** | PostgreSQL password for the Muxon DB user (must match the database). |
| **`keycloak-admin-password`** | Only if you run **bundled Keycloak** — admin password for the Keycloak container (separate from the OIDC client secret). |

On first start, the **bootstrap-initializer** runs before the backend services: it applies database migrations, writes encrypted settings, and records bootstrap state. It generates **per-service** config files under `/etc/muxon/` (for example `core-services-application.yaml`) and each service loads its own file via Spring Boot.

---

## Prerequisites

### All installation modes

- A computer or server with **64-bit x86** (typical PC or VM).
- **Network**: stable connectivity; for production, DNS and TLS certificates where users access the UI and APIs.
- **Firewall**: allow inbound **8080** (core-services HTTP) for Docker/Helm defaults; appliance may use NodePort/Ingress (see your values).

### Docker Compose

- **Docker Engine** 24+ and **Docker Compose** v2 (often `docker compose`).
- On Linux, your user should be in the `docker` group *or* you run commands with `sudo`.
- **OpenSSL** and **curl** — required by the Compose install script (`./install.sh`) to generate secrets and check health.

Check:

```bash
docker version
docker compose version
```

### Kubernetes and Helm

- A working **Kubernetes** cluster (1.28+ recommended).
- **`kubectl`** configured to talk to your cluster (`kubectl get nodes`).
- **Helm** 3.14+ (`helm version`).

You need permission to create namespaces, secrets, config maps, deployments, services, and (if using bundled Postgres) persistent volumes.

### VM appliance

- A **hypervisor** (Proxmox, VMware, KVM/QEMU, cloud VM, etc.).
- Ability to set **cloud-init user data** or to **SSH** into the VM after first boot.
- Enough RAM and disk for k3s and containers (e.g. **4 GB RAM**, **50 GB** disk minimum — match your appliance image requirements).

---

## Concepts

### `initial-config.yaml`

This YAML file describes:

- `muxon.instanceName` / `instanceId`
- System admin username (must match a user your IdP knows)
- IdP (`issuerUri`, `clientId`, and a placeholder for client secret — the real secret is in `muxon-oidc-secret`)
- Tenants
- JDBC URL and DB username (password comes from the secret file)

Templates:

- Docker: `compose/initial-config.yaml.template` inside the Compose install kit (or the `muxon-core` repository). The install script renders it to `initial-config.yaml`.
- Kubernetes: generated from Helm `values.yaml`, or overridden with `helm install ... --set-file bootstrap.initialConfigYaml=./initial-config.yaml`

### Secrets (files, not environment variables)

**Prefer files or Kubernetes Secret mounts** instead of putting passwords in compose `environment:` or plain `values.yaml`. This reduces accidental leaks via `docker inspect` or shell history.

---

## Option A — Docker Compose

The preferred trial path is the **versioned Compose install kit** attached to the OSS GitHub Release. The kit is self-contained: you do not need a git checkout. It includes `install.sh` (guided install) and `INSTALL.md` (the same steps, run by hand).

### Download the release kit

1. Open the [Muxon Core build releases](https://github.com/yorel-tech/muxon-core-build/releases).
2. On the release for your version (for example `v1.2.3`), download `muxon-compose-install-1.2.3.tar.gz` or `muxon-compose-install-1.2.3.zip`. The Helm chart `muxon-core-1.2.3.tgz` is attached to the same release. Weekly image builds do not publish this kit.
3. Unpack it and enter the directory:

```bash
tar -xzf muxon-compose-install-1.2.3.tar.gz
cd muxon-compose-install-1.2.3
```

The `VERSION` file in the kit is the image tag Compose uses unless you override it. That tag matches the `core-services` image published for the release.

### Guided install

Install Docker Engine 24+ and Compose v2 first if you have not already:

- **Ubuntu**: follow [Docker’s official install guide](https://docs.docker.com/engine/install/ubuntu/).
- **Other OS**: see [Docker Engine overview](https://docs.docker.com/engine/install/).

```bash
sudo systemctl enable --now docker
docker run --rm hello-world
./install.sh
```

Non-interactive example (bundled Postgres and Keycloak):

```bash
./install.sh --non-interactive \
  --profile internal \
  --admin-user admin@example.com
```

`./install.sh --help` lists flags. The script checks Docker, Compose v2, OpenSSL, and curl, creates `secrets/`, renders `initial-config.yaml` and the Keycloak realm import, starts Compose, and polls `http://127.0.0.1:8080/actuator/health`.

### Manual install

Follow **`INSTALL.md`** in the unpacked kit. It lists the same phases as the script: prerequisites, `./scripts/create-secrets.sh`, placeholder substitution, Compose up (internal or external), health check, and uninstall. Keep the shipped templates unchanged and write the runtime files the checklist names (`initial-config.yaml` at the kit root, and `compose/keycloak/realm-muxon-dev.runtime.json` for the bundled IdP).

### Developer path (git checkout)

From a `muxon-core` checkout, the same entrypoint and checklist live at the repository root (`./install.sh` and `INSTALL.md`). Compose paths are relative to `compose/`.

To build a kit the way the release workflow does:

```bash
cd muxon-core-build
VERSION=1.2.3 ./scripts/package-compose-install.sh
```

Equivalent manual commands from the `muxon-core` directory (or an unpacked kit):

```bash
./scripts/create-secrets.sh
./install.sh --render-only --non-interactive --admin-user admin@example.com
export MUXON_VERSION="$(cat VERSION)"
docker compose -f compose/muxon.yml --profile internal up -d
docker compose -f compose/muxon.yml ps
curl -sS http://127.0.0.1:8080/actuator/health
```

External database and IdP (no bundled Postgres or Keycloak):

```bash
./install.sh --non-interactive \
  --profile external \
  --admin-user admin@example.com \
  --issuer-uri https://idp.example.com/realms/muxon \
  --datasource-url jdbc:postgresql://db.example.com:5432/muxon
```

That runs `docker compose -f compose/muxon.yml up -d` without `--profile internal`.

---

## Option B — Kubernetes (Helm)

### Step 1 — Install `kubectl` and `helm`

Follow the official documentation for your OS:

- [kubectl install](https://kubernetes.io/docs/tasks/tools/)
- [Helm install](https://helm.sh/docs/intro/install/)

Confirm:

```bash
kubectl config current-context
helm version
```

### Step 2 — Create a namespace

```bash
kubectl create namespace muxon
```

### Step 3 — Create the Kubernetes Secret

From the directory where your secret **files** are:

```bash
kubectl create secret generic muxon-secrets \
  --namespace muxon \
  --from-file=muxon-passphrase=./secrets/muxon-passphrase \
  --from-file=muxon-oidc-secret=./secrets/muxon-oidc-secret \
  --from-file=muxon-db-password=./secrets/muxon-db-password \
  --from-file=keycloak-admin-password=./secrets/keycloak-admin-password
```

If you **do not** use bundled Keycloak, you can omit `keycloak-admin-password` **only if** `keycloak.enabled` is `false` in Helm values. Otherwise Keycloak needs that key.

### Step 4 — Choose internal vs external data services

**A) Bundled Postgres and Keycloak** — use a values fragment `my-values.yaml`:

```yaml
postgres:
  enabled: true
keycloak:
  enabled: true
  adminExistingSecret: muxon-secrets
  adminSecretKey: keycloak-admin-password

bootstrap:
  existingSecret: muxon-secrets
  initialConfig:
    issuerUri: "http://keycloak:8085/realms/muxon-local"
    datasourceUrl: "jdbc:postgresql://postgres:5432/muxon"
```

**B) External Postgres / IdP** — disable bundled services and point URLs at your endpoints:

```yaml
postgres:
  enabled: false
keycloak:
  enabled: false

bootstrap:
  existingSecret: muxon-secrets
  initialConfig:
    issuerUri: "https://idp.example.com/realms/myrealm"
    datasourceUrl: "jdbc:postgresql://db.example.com:5432/muxon"
```

Adjust `image.repository` / `image.tag` to match the registry where your `core-services` image is stored.

### Step 5 — Install the chart

From the `muxon-core` repo:

```bash
helm upgrade --install muxon ./deploy/helm/muxon-core \
  --namespace muxon \
  -f my-values.yaml \
  --set-file bootstrap.initialConfigYaml=./initial-config.yaml
```

If you omit `--set-file`, Helm builds `initial-config.yaml` from `bootstrap.initialConfig` in `values.yaml`.

### Step 6 — Verify

```bash
kubectl -n muxon get pods
kubectl -n muxon logs deploy/core-services -c bootstrap
kubectl -n muxon logs deploy/core-services -c core-services --tail=100
kubectl -n muxon port-forward svc/core-services 8080:8080
curl -sS http://127.0.0.1:8080/actuator/health
```

The deployment uses an **init container** to run bootstrap; the main container sets **`MUXON_SKIP_BOOTSTRAP=1`** so bootstrap is not run twice.

---

## Option C — VM appliance (k3s)

The appliance image includes **k3s**, preloaded container images, and a packaged **Helm chart** (`.tgz` under `/opt/muxon/charts/`). On first boot, **`muxon-firstboot.service`** imports images, creates the `muxon` namespace, creates **`muxon-secrets`**, and runs **`helm install`**.

### Operator cloud-init (recommended for automation)

Cloud-init configuration is a **`#cloud-config` YAML** file. You usually inject **files** using `write_files`, not loose “key=value” pairs.

- **Proxmox**: VM → **Cloud-Init** tab → **User data** → paste the YAML.
- **VMware**: VM settings → **vApp Options** / **Advanced** → **Properties** → user data (often **base64**-encode the whole YAML).
- **KVM / QEMU**: build a NoCloud seed ISO (`cloud-localds`) with your `user-data` file.
- **Public clouds**: paste into the instance **User data** field at launch.

**Example file (edit before use):** in the **`muxon-build`** repository, path  
`packer/cloud-init/operator-user-data.example.yaml` (used when building the appliance; copy it into your hypervisor’s user-data field).

Encode a secret for `encoding: b64`:

```bash
echo -n 'your-secret-value' | base64 -w0   # GNU coreutils
# macOS: echo -n 'your-secret-value' | base64
```

If you **omit** secret files, the first-boot script **generates** random files under `/etc/muxon/secrets/`. Retrieve them with `sudo cat` after boot.

### Interactive wizard (alternative)

On an appliance image that includes the script:

```bash
sudo muxon-setup-wizard.sh
```

Or copy `muxon-build/scripts/appliance-setup-wizard.sh` from the platform build repo onto the VM and run it with `sudo bash`.

Then reboot or start the installer:

```bash
sudo systemctl start muxon-firstboot.service
```

### Monitor first boot

```bash
sudo journalctl -u muxon-firstboot.service -f
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
sudo k3s kubectl -n muxon get pods
```

### Build metadata on the VM

```bash
cat /etc/muxon/build.env
```

---

## Upgrading

- **Docker**: pull newer images, `docker compose up -d`, read release notes for DB migrations (Flyway runs on bootstrap when applicable).
- **Helm**: `helm upgrade` with the same secret and config strategy; use `--set-file` if you manage `initial-config.yaml` externally.
- **Appliance**: build a new image from `muxon-build` and follow your migration runbook (backup Postgres first).

---

## Uninstalling

- **Docker Compose**: `docker compose -f compose/muxon.yml down` (add `-v` to remove volumes — **destroys database data**).
- **Helm**: `helm uninstall muxon -n muxon` (and delete PVCs if Postgres was enabled).
- **Appliance**: remove the VM or reinstall the OS image.

---

## Troubleshooting

| Symptom | Things to check |
|--------|------------------|
| Bootstrap fails on DB connection | JDBC URL, network policy, secret `muxon-db-password` matches Postgres. |
| `OIDC` / login errors | `issuerUri` and `clientId` / `muxon-oidc-secret` match the IdP client. |
| Keycloak never ready | Memory/CPU; check `kubectl logs -n muxon deploy/keycloak`. |
| Docker Compose and “profile internal” | Postgres/Keycloak only start with `--profile internal`. |
| Permission denied on secrets | Files under `secrets/` should be readable only by you (`chmod 600`). |

For developers, see also [initial-bootstrap.md](../../muxon-core/dev-docs/initial-bootstrap.md).
