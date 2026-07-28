# Content Library — Publishing Guide

This guide covers how to publish VM templates and scripts to the Muxon content library, including the prerequisites that must be satisfied for guest customization to work correctly.

---

## Overview

The content library stores:

| Content type | Description |
|---|---|
| `vm-template` | Golden disk images (QCOW2, raw, VMDK, OVA) from which VMs are deployed |
| `iso` | Bootable or driver ISO images |
| `script` | Shell / PowerShell / Python scripts for pre/post-customization |

---

## Publishing a VM template

### 1. Prepare the image

All VM templates must satisfy the baseline prerequisites for guest customization before being published. These prerequisites differ by OS family.

#### Linux prerequisites

| Item | Requirement |
|---|---|
| `cloud-init` | Must be installed and enabled. Version 21.1+ recommended. |
| NoCloud datasource | `/etc/cloud/cloud.cfg.d/99_muxon.cfg` must contain `datasource_list: [NoCloud, None]` |
| `qemu-guest-agent` | Must be installed and the systemd unit enabled. |
| VirtIO drivers | Must be available in the kernel/initramfs (standard on Ubuntu 20.04+, RHEL 8+). |
| Clean instance-id | `cloud-init clean --logs --seed` must be run before shutdown. |

**Quick verification** (run inside the template VM before shutdown):

```bash
# Verify cloud-init is installed
cloud-init --version

# Verify the NoCloud datasource is configured
grep -r NoCloud /etc/cloud/

# Verify qemu-guest-agent is enabled
systemctl is-enabled qemu-guest-agent

# Clean cloud-init state
sudo cloud-init clean --logs --seed
sudo shutdown -h now
```

#### Windows prerequisites

| Item | Requirement |
|---|---|
| VirtIO drivers | `virtio-net`, `virtio-scsi`, `virtio-balloon`, `virtio-serial` must be installed from the [virtio-win ISO](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/). |
| QEMU Guest Agent | `qemu-ga-x64.msi` must be installed; service set to `Automatic` start. |
| PowerShell | Version 5.x or later (built-in on Windows 10 / Server 2016+). |
| Sysprep | Image must be generalized: `sysprep.exe /generalize /oobe /shutdown`. |

**Quick verification** (run inside the Windows template VM before sysprep):

```powershell
# Verify virtio-net driver is present
Get-NetAdapter | Where-Object { $_.DriverDescription -like "*VirtIO*" }

# Verify QGA service
Get-Service QEMU-GA | Select-Object Status, StartType

# Verify PowerShell version
$PSVersionTable.PSVersion
```

Then run sysprep:

```bat
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

> **Important:** After sysprep shuts down the VM, do **not** boot it again before exporting. Booting a sysprepped image without an `Autounattend.xml` will cause Windows to prompt for input and may invalidate the generalized state.

### 2. Export the disk image

Export the VM disk as QCOW2 (recommended for KVM/Proxmox):

```bash
# Libvirt / KVM
virsh dumpxml <vm-name>   # note the disk path
qemu-img convert -O qcow2 /var/lib/libvirt/images/my-template.qcow2 ./my-template-export.qcow2

# Proxmox
qm exportdisk <vmid> <disk> ./my-template-export.qcow2
```

### 3. Upload to the content library

Use the Muxon API or UI:

```
POST /content-library/items
Content-Type: multipart/form-data

name:              "ubuntu-2404-base"
contentType:       vm-template
providerType:      libvirt          # or proxmox
file:              <my-template-export.qcow2>
spec: {
  "metadata": {
    "osFamily": "linux",
    "osName": "Ubuntu 24.04 LTS",
    "diskFormat": "qcow2"
  },
  "customization": {
    "type": "cloudInit",
    "osFamily": "linux",
    "defaults": {
      "timezone": "UTC",
      "dnsServers": []
    }
  }
}
```

For Windows:

```
POST /content-library/items
Content-Type: multipart/form-data

name:        "windows-server-2022-base"
contentType: vm-template
providerType: libvirt
file:        <windows-server-2022.qcow2>
spec: {
  "metadata": {
    "osFamily": "windows",
    "osName": "Windows Server 2022",
    "diskFormat": "qcow2"
  },
  "customization": {
    "type": "sysprep",
    "osFamily": "windows",
    "defaults": {
      "timezone": "Pacific Standard Time"
    }
  }
}
```

### 4. Validate template availability

After upload, the item status must be `AVAILABLE` before it can be deployed:

```
GET /content-library/items/{id}
→ { "status": "AVAILABLE", ... }
```

---

## Publishing a script item

Scripts can be stored in the content library and referenced by `preScriptItemIds` / `postScriptItemIds` at deploy time.

### Linux shell scripts

```
POST /content-library/items
Content-Type: multipart/form-data

name:        "install-datadog-agent"
contentType: script
file:        install-datadog.sh
spec: {
  "metadata": {
    "osFamily": "linux",
    "interpreter": "bash",
    "description": "Installs and configures the Datadog agent"
  }
}
```

### Windows PowerShell scripts

```
POST /content-library/items
Content-Type: multipart/form-data

name:        "configure-windows-firewall"
contentType: script
file:        configure-firewall.ps1
spec: {
  "metadata": {
    "osFamily": "windows",
    "interpreter": "powershell",
    "description": "Configures Windows Firewall rules for corporate policy"
  }
}
```

The platform stores the SHA-256 checksum of the script at upload time. At deploy time, the `ScriptFetcher` re-validates the checksum before embedding the script into the seed ISO.

---

## Guest customization prerequisites checklist

Use this checklist when submitting a new template to the content library.

### Linux

- [ ] `cloud-init` version 21.1+ installed
- [ ] `/etc/cloud/cloud.cfg.d/99_muxon.cfg` sets `datasource_list: [NoCloud, None]`
- [ ] `qemu-guest-agent` installed, service enabled
- [ ] VirtIO drivers available in kernel
- [ ] `cloud-init clean --logs --seed` run before final shutdown
- [ ] Template `spec.customization.type` = `cloudInit`
- [ ] Template `spec.customization.osFamily` = `linux`

### Windows

- [ ] VirtIO drivers installed (net, scsi, balloon, serial)
- [ ] `qemu-ga-x64.msi` installed, QEMU-GA service set to `Automatic`
- [ ] PowerShell 5.x+ available
- [ ] Image sysprepped with `/generalize /oobe /shutdown`
- [ ] Template `spec.customization.type` = `sysprep`
- [ ] Template `spec.customization.osFamily` = `windows`

---

## Related documentation

- [Guest Customization — Template Preparation Guide](guest-customization-templates.md): Detailed per-OS preparation steps.
- [Guest Customization — User Guide](../user-guide/guest-customization.md): How to request customization when deploying a VM.
