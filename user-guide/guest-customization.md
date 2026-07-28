# Guest Customization

Guest customization lets you fully configure a VM at deploy time: hostname, IP addresses, domain membership, users, packages, and arbitrary scripts — all without logging into the guest after boot.

---

## Prerequisites

Before deploying a VM with customization, ensure:

| Requirement | Linux | Windows |
|---|---|---|
| `cloud-init` installed in template | Required | N/A |
| `qemu-guest-agent` installed | Required (status reporting) | Required (status reporting) |
| VirtIO drivers installed | Usually built-in | Required (must install from virtio-win ISO) |
| Image sysprepped | N/A | Required |

See [Template Preparation Guide](../admin-guide/guest-customization-templates.md) for detailed instructions on preparing template images.

---

## Requesting customization at deploy time

Pass a `customization` block in the `POST /vms` request body:

```json
{
  "templateItemId": "<template-uuid>",
  "computeNodeId":  "<node-uuid>",
  "name": "web-prod-01",
  "customization": {
    "osFamily": "linux",
    "hostname": "web-prod-01",
    "nics": [
      {
        "macAddress": "52:54:00:aa:bb:01",
        "ipAllocation": "static",
        "ipAddress":   "10.1.2.50",
        "prefix":      24,
        "gateway":     "10.1.2.1",
        "dnsServers":  ["10.1.0.1", "10.1.0.2"]
      }
    ],
    "linux": {
      "timezone": "America/Los_Angeles",
      "dnsSearch": ["corp.example.com"],
      "users": [
        {
          "name":              "adminuser",
          "sshAuthorizedKeys": ["ssh-ed25519 AAAA..."],
          "sudo":              "ALL=(ALL) NOPASSWD:ALL",
          "shell":             "/bin/bash"
        }
      ],
      "packages": ["nginx", "git"],
      "postScriptItemIds": ["<script-item-uuid>"]
    }
  }
}
```

For DHCP, omit `ipAddress`, `prefix`, and `gateway`:

```json
"nics": [
  { "macAddress": "52:54:00:aa:bb:01", "ipAllocation": "dhcp" }
]
```

### Windows example

```json
{
  "customization": {
    "osFamily": "windows",
    "hostname": "WIN-WEBPROD01",
    "nics": [
      {
        "macAddress": "52:54:00:aa:bb:02",
        "ipAllocation": "static",
        "ipAddress":   "10.1.2.51",
        "prefix":      24,
        "gateway":     "10.1.2.1",
        "dnsServers":  ["10.1.0.1"]
      }
    ],
    "windows": {
      "adminPassword":     "<see Password Security section>",
      "productKey":        "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX",
      "timezone":          "Pacific Standard Time",
      "uiLanguage":        "en-US",
      "inputLocale":       "en-US",
      "joinDomain":        "CORP",
      "domainFqdn":        "corp.example.com",
      "domainJoinUser":    "SVC_JOIN@corp.example.com",
      "domainJoinPassword": "<see Password Security section>",
      "postScriptItemIds": ["<ps-script-item-uuid>"]
    }
  }
}
```

---

## Custom scripts

Scripts run at customization time can be provided in two ways:

### Inline scripts

Embed script content directly in the request body:

```json
"linux": {
  "bootcmd": [
    "echo 'nameserver 10.1.0.1' >> /etc/resolv.conf"
  ],
  "runcmd": [
    "systemctl enable nginx",
    "systemctl start nginx"
  ]
}
```

### Content library scripts

Reference a script item by ID from the content library. Scripts must be published with `contentType: script` first.

```json
"linux": {
  "preScriptItemIds":  ["<uuid-of-pre-script>"],
  "postScriptItemIds": ["<uuid-of-post-script>"]
}
```

The platform fetches the script content from the content library at deploy time, embeds it into the seed ISO, and verifies the SHA-256 checksum before running. Scripts are **never transmitted to the guest in plain text** — they are baked into the ephemeral seed ISO, which is deleted once customization completes.

#### When scripts run

| Field | OS | Execution stage |
|---|---|---|
| `linux.preScriptItemIds` | Linux | `bootcmd` — before network is configured |
| `linux.runcmd` | Linux | `runcmd` — after network/users/packages |
| `linux.postScriptItemIds` | Linux | `final` — last stage, all services up |
| `windows.preScriptItemIds` | Windows | `specialize` pass — before OOBE |
| `windows.firstLogonCommands` | Windows | `oobeSystem` — on first login |
| `windows.postScriptItemIds` | Windows | Appended to `firstLogonCommands` — after domain join |

---

## Password security

Passwords (`adminPassword`, `domainJoinPassword`, `users[].password`) are **always write-only** — they are never returned in any GET response.

### At rest

All sensitive fields in the `customization` column are AES-256-GCM field-level encrypted in the database. Each field uses a unique random 12-byte IV. The key is provided via environment variable at runtime and never stored in the database.

### In transit

- The API layer accepts passwords in clear text over TLS (HTTPS only).
- Inside the seed ISO (which lives only on the worker host filesystem, not in the database), passwords are encoded only as required by the OS (e.g., the Windows admin password is Base64-encoded per the Microsoft sysprep spec).
- The seed ISO path is set to `0600` permissions and its parent directory to `0700`.
- After customization completes (or fails), the seed ISO is **immediately deleted from disk** and the seed path is cleared from the database.

### Audit log

Passwords are redacted before any log or audit event is emitted. Only the field name is logged (e.g., `"adminPassword": "[REDACTED]"`).

---

## Customization status

Query the VM to check progress:

```
GET /vms/{id}
```

The response includes:

```json
{
  "id": "...",
  "customization_status": {
    "phase":     "IN_PROGRESS",
    "subPhase":  "SCRIPTS_POST",
    "message":   "Running post-customization scripts",
    "startedAt": "2026-04-22T10:00:00Z",
    "updatedAt": "2026-04-22T10:05:30Z"
  }
}
```

### Phase lifecycle

```
NONE → PENDING → SEED_BUILT → WAITING_AGENT → IN_PROGRESS → COMPLETE
                                                           └→ FAILED
```

| Phase | Description |
|---|---|
| `NONE` | No customization requested |
| `PENDING` | Customization spec persisted; worker not yet picked up the job |
| `SEED_BUILT` | Seed ISO created and attached to VM |
| `WAITING_AGENT` | VM booting; waiting for QEMU Guest Agent to become reachable |
| `IN_PROGRESS` | Guest is actively running customization steps |
| `COMPLETE` | All steps done; seed ISO detached and deleted |
| `FAILED` | An error occurred; seed ISO detached and deleted |

### Linux sub-phases (`subPhase`)

| Sub-phase | Description |
|---|---|
| `NETWORK_CONFIG` | cloud-init `network` stage: configuring NICs |
| `CONFIG` | cloud-init `config` stage: users, packages, files |
| `SCRIPTS_PRE` | cloud-init `bootcmd` / pre-scripts running |
| `SCRIPTS_POST` | cloud-init `final` stage: runcmd and post-scripts |

### Windows sub-phases

Windows does not report granular sub-phases via QGA. The platform transitions directly to `IN_PROGRESS` once the guest is reachable, and to `COMPLETE` once the expected hostname is reported back by QGA.

---

## Post-customization expectations

After customization reaches `COMPLETE`:

- The VM's `hostname` field is updated to the value reported by the guest agent.
- The VM's `ip_addresses` field is updated from QGA network interface data.
- The seed ISO is detached and the backing file deleted.
- `customization_seed_path` is cleared in the database.
- All passwords are removed from the encrypted `customization` column (the secrets are zeroed out).

The VM is fully operational. You can SSH/RDP in using the credentials provided in the customization spec.

---

## Behavior during and after migration

When a VM is live-migrated to a different compute node:

- The seed ISO is **not** migrated. It is a temporary file on the source node.
- If the customization is still in progress at migration time, the seed ISO is already attached to the VM; libvirt/Proxmox will migrate the VM's attached disk metadata along with the domain definition.
- The `GuestCustomizationMonitor` runs on the **worker that originally created the VM**. It will continue polling QGA (routed through the provider layer) even after migration.
- If the VM is cloned, the clone does **not** inherit the `customization` spec or the `customization_status`. The operator must re-deploy with a new customization spec if the clone needs to be configured differently.

---

## Troubleshooting

| Symptom | Likely cause | Resolution |
|---|---|---|
| Phase stuck at `WAITING_AGENT` | `qemu-guest-agent` not installed or not started in template | Verify template prerequisites; check `qemu-guest-agent` service status |
| Phase stuck at `IN_PROGRESS` | Cloud-init failure (Linux) or sysprep failure (Windows) | Check cloud-init logs: `/var/log/cloud-init.log`, Windows Event Viewer (Setup log) |
| Phase reaches `FAILED` | Script error, domain join failure, bad password | Check `message` field in `customization_status` for details |
| Network not configured | Static NIC missing `prefix` or `gateway` | Ensure all static NIC fields are populated |
| Windows hostname not updated | Hostname > 15 chars for Windows | Use at most 15 characters for Windows hostnames |
