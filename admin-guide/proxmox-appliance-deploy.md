# Deploy the Muxon appliance on Proxmox (QCOW2 + Cloud-Init)

This guide shows how to deploy the **Muxon appliance QCOW2** on **Proxmox VE** and pass configuration via **Cloud-Init**.

The appliance boots an embedded **k3s** cluster and runs a first-boot initializer that:

- Imports preloaded container images (if present)
- Installs the packaged Helm chart under `/opt/muxon/charts/`
- Reads `/etc/muxon/initial-config.yaml` (if provided) or writes a default
- Reads secret files under `/etc/muxon/secrets/` (if provided) or generates them

---

## Prerequisites

- A Proxmox node (or cluster) with CLI access (`ssh root@<node>`)
- Storage available for VM disks:
  - Common choices: `local-lvm` (LVM-thin) or `local` (directory)
- The appliance QCOW2 file (for example `muxon-oss-<version>.qcow2`)

Optional but recommended:

- A Proxmox storage that supports **snippets** (often `local`) for custom Cloud-Init user-data

---

## 1) Upload the QCOW2 to the Proxmox node

From your workstation:

```bash
scp dist/muxon-oss-0.1.0.qcow2 root@<proxmox-node>:/root/
```

---

## 2) Create a VM shell (CPU, RAM, NIC)

Run on the Proxmox node:

```bash
VMID=120
STORAGE=local-lvm

qm create $VMID \
  --name muxon-appliance \
  --memory 8192 \
  --cores 4 \
  --net0 virtio,bridge=vmbr0

qm set $VMID --scsihw virtio-scsi-pci
```

Notes:

- Increase RAM/CPU if you enable bundled Keycloak/Postgres and run larger workloads.
- `virtio` networking is recommended for performance.

---

## 3) Import the QCOW2 as the VM’s disk

```bash
qm importdisk $VMID /root/muxon-oss-0.1.0.qcow2 $STORAGE
qm set $VMID --scsi0 $STORAGE:vm-$VMID-disk-0
qm set $VMID --boot order=scsi0
```

---

## 4) Add a Cloud-Init drive

```bash
qm set $VMID --ide2 $STORAGE:cloudinit
qm set $VMID --serial0 socket --vga serial0
```

---

## 5) Configure basic Cloud-Init (SSH + networking)

### SSH user and key

```bash
qm set $VMID --ciuser muxon
qm set $VMID --sshkeys /root/id_rsa.pub
```

### Networking

DHCP:

```bash
qm set $VMID --ipconfig0 ip=dhcp
```

Static IP (example):

```bash
qm set $VMID --ipconfig0 ip=192.168.1.50/24,gw=192.168.1.1
```

---

## 6) Pass Muxon config + secrets via Cloud-Init “snippets”

### 6.1 Create a Cloud-Init user-data snippet

On the Proxmox node, create:

`/var/lib/vz/snippets/muxon-user-data.yml`

```yaml
#cloud-config
write_files:
  # --- Secrets (if you provide them, the appliance will use these instead of generating) ---
  - path: /etc/muxon/secrets/muxon-passphrase
    permissions: "0600"
    content: |
      REPLACE_ME_muxon_passphrase

  - path: /etc/muxon/secrets/muxon-oidc-secret
    permissions: "0600"
    content: |
      REPLACE_ME_muxon_oidc_secret

  - path: /etc/muxon/secrets/muxon-db-password
    permissions: "0600"
    content: |
      REPLACE_ME_muxon_db_password

  - path: /etc/muxon/secrets/keycloak-admin-password
    permissions: "0600"
    content: |
      REPLACE_ME_keycloak_admin_password

  # --- Initial config ---
  - path: /etc/muxon/initial-config.yaml
    permissions: "0644"
    content: |
      muxon:
        instanceName: "appliance"
        instanceId: 1
        system:
          systemAdminUserName: "admin@muxon.local"
          idp:
            name: "default-idp"
            protocol: "OIDC"
            metadata:
              issuerUri: "http://keycloak:8085/realms/muxon-local"
              clientId: "muxon-api"
              clientSecret: "provide-in-secrets"
        tenants:
          - name: "default"
            idpName: "default-idp"
            tenantAdminUserName: "admin@muxon.local"
        datasource:
          url: jdbc:postgresql://postgres:5432/muxon
          username: muxon
          password: "ignored"
          driver-class-name: org.postgresql.Driver
```

Notes:

- `muxon-oidc-secret` is the **OIDC client secret** for the Keycloak client `muxon-api` (not the Keycloak admin password).
- The datasource password placeholder is not trusted at runtime; the DB password comes from `muxon-db-password`.

### 6.2 Attach the snippet to the VM

```bash
qm set $VMID --cicustom "user=local:snippets/muxon-user-data.yml"
qm cloudinit update $VMID
```

To inspect what Proxmox will feed the VM:

```bash
qm cloudinit dump $VMID user
```

---

## 7) Boot and verify

Start the VM:

```bash
qm start $VMID
```

Follow first boot via serial console (recommended):

```bash
qm terminal $VMID
```

After boot, SSH in:

```bash
ssh muxon@<vm-ip>
```

Useful checks:

```bash
sudo systemctl status muxon-firstboot.service --no-pager
sudo journalctl -u muxon-firstboot.service -b --no-pager

sudo ls -l /etc/muxon/
sudo ls -l /etc/muxon/secrets/

sudo k3s kubectl get nodes
sudo k3s kubectl -n muxon get pods
```

---

## Troubleshooting

### Cloud-Init did not apply

Check:

```bash
sudo cloud-init status --long
sudo tail -n 200 /var/log/cloud-init-output.log
```

### Keycloak / realm mismatch

If you use the embedded Keycloak, ensure your `issuerUri` realm matches the realm imported by the chart (for example `muxon-local`).

### Secrets recovery

If the appliance generated secrets automatically, they are stored under:

- `/etc/muxon/secrets/`

and also copied into Kubernetes as:

- Secret `muxon-secrets` in namespace `muxon`

---

## Related documentation

- [Install Muxon Core](install-core.md) (VM appliance section)
- [Guest Customization — Template Preparation Guide](guest-customization-templates.md)

