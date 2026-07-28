# Guest Customization — Template Preparation Guide

This document describes the prerequisites that must be baked into every VM template image before it is published to the Muxon content library. Templates that meet these requirements can be fully customized at deploy time (hostname, IP, domain, users, scripts, etc.) without manual intervention.

---

## Linux templates

### Required packages

| Package | Purpose |
|---|---|
| `cloud-init` | Cloud-init performs all customization: network, users, SSH keys, packages, runcmd scripts. |
| `qemu-guest-agent` | QEMU Guest Agent (QGA) — the KVM equivalent of VMware Tools. Used by the control plane for IP/hostname telemetry and graceful shutdown. **Not required for customization itself, but required for post-boot status reporting.** |

#### Ubuntu / Debian
```bash
apt-get update
apt-get install -y cloud-init qemu-guest-agent
systemctl enable qemu-guest-agent
```

#### RHEL / Rocky / AlmaLinux / CentOS Stream
```bash
dnf install -y cloud-init qemu-guest-agent
systemctl enable qemu-guest-agent
```

#### SUSE / OpenSUSE
```bash
zypper install -y cloud-init qemu-guest-agent
systemctl enable qemu-guest-agent
```

### Cloud-init datasource configuration

Cloud-init must be configured to read from the `NoCloud` datasource (the seed ISO labeled `CIDATA`). Create the file below **inside the template image**:

```bash
cat > /etc/cloud/cloud.cfg.d/99_muxon.cfg << 'EOF'
datasource_list: [NoCloud, None]
EOF
```

This ensures cloud-init always reads from the `CIDATA` ISO and does not attempt to contact a metadata server.

### Final cleanup before shutdown (mandatory)

Before shutting down the template to export/publish it, run:

```bash
cloud-init clean --logs --seed
```

This removes the previous instance-id so the first boot of any cloned VM picks up a fresh `meta-data` instance-id from the seed ISO. **If you skip this step, cloud-init will not re-run on deployed VMs.**

### VirtIO drivers

All modern Linux distro kernels include VirtIO drivers in the kernel or initramfs. No manual installation is required for Ubuntu 20.04+, RHEL 8+, or any kernel ≥ 4.15.

### Template spec customization block

When publishing the template to the content library, set:

```yaml
spec:
  customization:
    type: cloudInit
    osFamily: linux
    defaults:
      timezone: UTC          # optional default
      dnsServers: []         # optional default DNS
```

---

## Windows templates

### Required components

| Component | Source | Purpose |
|---|---|---|
| VirtIO drivers | [virtio-win ISO](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/) | virtio-net, virtio-scsi, virtio-balloon, virtio-serial — required for KVM/Proxmox. |
| QEMU Guest Agent | Bundled in virtio-win ISO: `virtio-win/guest-agent/qemu-ga-x64.msi` | Post-boot IP/hostname telemetry. Set the service to `Automatic` start. |
| PowerShell 5.x+ | Built-in on Windows 10 / Server 2016+ | Required for `FirstLogonCommands` execution. |

### Installation steps

1. **Install VirtIO drivers:**
   - Boot the Windows image from the installation ISO.
   - Attach the `virtio-win` ISO as a second CD-ROM.
   - Open Device Manager and install drivers for all unknown devices, pointing to the virtio-win ISO.
   - Alternatively, run the `virtio-win-gt-x64.msi` installer from the virtio-win ISO.

2. **Install QEMU Guest Agent:**
   ```bat
   msiexec /i D:\guest-agent\qemu-ga-x64.msi /quiet
   sc config QEMU-GA start= auto
   net start QEMU-GA
   ```

3. **Sysprep** (mandatory before publishing):
   ```bat
   C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown /unattend:C:\path\to\baseline-unattend.xml
   ```
   After shutdown, the disk image is ready to publish as a template.

### Baseline Autounattend.xml

The template image should be sysprepped with a minimal unattend that only handles the OOBE skip and locale. Muxon will inject a full `Autounattend.xml` at deploy time via the seed ISO:

```xml
<?xml version="1.0" encoding="utf-8"?>
<unattend xmlns="urn:schemas-microsoft-com:unattend">
  <settings pass="oobeSystem">
    <component name="Microsoft-Windows-Shell-Setup"
               processorArchitecture="amd64"
               publicKeyToken="31bf3856ad364e35"
               language="neutral"
               versionScope="nonSxS">
      <OOBE>
        <HideEULAPage>true</HideEULAPage>
        <HideLocalAccountScreen>true</HideLocalAccountScreen>
        <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
        <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
        <ProtectYourPC>3</ProtectYourPC>
        <SkipMachineOOBE>true</SkipMachineOOBE>
        <SkipUserOOBE>true</SkipUserOOBE>
      </OOBE>
    </component>
  </settings>
</unattend>
```

### Template spec customization block

```yaml
spec:
  customization:
    type: sysprep
    osFamily: windows
    defaults:
      timezone: Pacific Standard Time
```

---

## Content library script items

Scripts to be executed during customization can be stored as **`script` content items** in the content library instead of being inlined in every deployment request. This keeps templates immutable and makes scripts independently versioned.

### Creating a script item

1. Navigate to your content library in the Muxon UI (or use the API).
2. Create a new item with `contentType: script`.
3. Upload the script file (shell, PowerShell, Python, etc.).
4. Note the `id` (UUID) — this is the `preScriptItemId` or `postScriptItemId` used at deploy time.

**Recommended metadata fields:**

| Field | Example value |
|---|---|
| `osFamily` | `linux` or `windows` |
| `description` | "Install datadog agent" |
| `version_label` | "1.2.0" |

### Execution stages by OS

| Stage | OS | When it runs |
|---|---|---|
| `preScriptItemIds` | Linux | `bootcmd` — before network is configured |
| `preScriptItemIds` | Windows | `specialize` pass — before OOBE, after domain join |
| `postScriptItemIds` | Linux | `runcmd` / `final` — after all config (network, users, packages) |
| `postScriptItemIds` | Windows | `firstLogonCommands` — after domain join and OOBE |

### Validation at template publish time

When a template is published to the content library, the platform validates:

- `spec.customization.type` is set and matches `osFamily` (`cloudInit` for linux, `sysprep` for windows).
- `spec.customization.osFamily` is present.
- (Linux) `cloud-init` is detectable via the `template.spec.metadata.osFamily=linux` tag; the admin is responsible for running `cloud-init clean` before export.
- (Windows) The image must be sysprepped; the platform cannot verify this directly — it is the admin's responsibility.

---

## Checklist

### Linux template checklist

- [ ] `cloud-init` installed and enabled
- [ ] `/etc/cloud/cloud.cfg.d/99_muxon.cfg` created with `datasource_list: [NoCloud, None]`
- [ ] `qemu-guest-agent` installed and enabled
- [ ] `cloud-init clean --logs --seed` run before shutdown
- [ ] Template spec `customization.type=cloudInit`, `customization.osFamily=linux`

### Windows template checklist

- [ ] VirtIO drivers installed (virtio-net, virtio-scsi, virtio-balloon, virtio-serial)
- [ ] QEMU Guest Agent MSI installed, service set to Automatic
- [ ] PowerShell 5.x+ available
- [ ] Image sysprepped with `/generalize /oobe /shutdown`
- [ ] Template spec `customization.type=sysprep`, `customization.osFamily=windows`
