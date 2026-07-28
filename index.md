---
title: Infron Documentation
audience: public
---

# Infron Documentation

Welcome to the Infron documentation for **Muxon Core** (OSS) and **Infron Nexus** (Enterprise).

## What you can do here

- Learn key concepts and architecture
- Install and operate the platform
- Use the UI and API to provision and manage resources

## Editions

- **Core (OSS):** baseline resource orchestration
- **Nexus (Enterprise):** enterprise extensions (overcommit policies, advanced content library, multi-IdP, and more)

---

## Concepts

Foundational ideas and platform architecture.

| Document | Topic |
|----------|-------|
| [Overview](concepts/overview.md) | What Infron is and main building blocks |
| [Platform Architecture](concepts/platform-architecture.md) | Control plane services and data flow |
| [Editions — Core vs Nexus](concepts/editions-core-vs-nexus.md) | Feature and deployment differences |
| [Tenancy and Identity](concepts/tenancy-and-identity.md) | Tenants, projects, and identity |
| [Providers and Clusters](concepts/providers-and-clusters.md) | Infrastructure providers |
| [Resources and Classes](concepts/resources-and-classes.md) | VM classes, networks, storage |
| [API and Extensibility](concepts/api-and-extensibility.md) | REST API and extension points |

---

## Admin Guide

Install, configure, and operate the platform.

| Document | Topic |
|----------|-------|
| [Overview](admin-guide/overview.md) | Audience and documentation map |
| [Install Core](admin-guide/install-core.md) | Docker, Kubernetes/Helm, appliance |
| [Install Nexus](admin-guide/install-nexus.md) | Enterprise Helm deployment |
| [Proxmox Appliance Deploy](admin-guide/proxmox-appliance-deploy.md) | Proxmox-specific appliance |
| [Configuration](admin-guide/configuration.md) | System settings |
| [Identity and Access](admin-guide/identity-and-access.md) | IdP, roles, RBAC |
| [Providers](admin-guide/providers.md) | Register hypervisor backends |
| [Node Clusters](admin-guide/node-clusters.md) | Cluster inventory |
| [Datacenters](admin-guide/datacenters.md) | Datacenter topology |
| [Networking](admin-guide/networking.md) | Networks and fabrics |
| [Storage](admin-guide/storage.md) | Storage classes and volumes |
| [VM Classes](admin-guide/vm-classes.md) | Compute profiles |
| [Tenants and Projects](admin-guide/tenants-and-projects.md) | Multi-tenancy setup |
| [Content Library Publish](admin-guide/content-library-publish-guide.md) | Template publishing |
| [Guest Customization Templates](admin-guide/guest-customization-templates.md) | Cloud-init / sysprep templates |
| [Monitoring and Logging](admin-guide/monitoring-and-logging.md) | Observability |
| [Backup and DR](admin-guide/backup-and-dr.md) | Backup strategy |
| [Upgrades and Migrations](admin-guide/upgrades-and-migrations.md) | Version upgrades |
| [Hardening and Compliance](admin-guide/hardening-and-compliance.md) | Security posture |
| [Troubleshooting](admin-guide/troubleshooting.md) | Common operator issues |

---

## User Guide

Tenant workflows in the UI and API.

| Document | Topic |
|----------|-------|
| [Overview](user-guide/overview.md) | What tenant users can do |
| [Onboarding](user-guide/onboarding.md) | Getting started |
| [Projects and Access](user-guide/projects-and-access.md) | Project scope and permissions |
| [Create VM](user-guide/create-vm.md) | Provision a workload |
| [Manage VM](user-guide/manage-vm.md) | Lifecycle operations |
| [Guest Customization](user-guide/guest-customization.md) | OS customization |
| [Images and Templates](user-guide/images-and-templates.md) | Catalog items |
| [Storage Operations](user-guide/storage-operations.md) | Volumes and snapshots |
| [Networking Operations](user-guide/networking-operations.md) | Attach networks |
| [Quotas and Usage](user-guide/quotas-and-usage.md) | Limits and consumption |
| [Automation and API](user-guide/automation-and-api.md) | API and automation |
| [Troubleshooting](user-guide/troubleshooting.md) | End-user issues |

---

## For developers

Internal setup, architecture, and build docs live in [`muxon-core/dev-docs/`](../muxon-core/dev-docs/).
