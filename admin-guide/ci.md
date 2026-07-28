# CI/CD

Muxon uses **separate GitHub repositories** for each package. Locally they are sibling checkouts under `development/`.

| Repo | Visibility |
|------|------------|
| `muxon-core` | Public |
| `muxon-core-build` | Public |
| `muxon-web` | Public (common) |
| `muxon-docs` | Public (common) |
| `muxon-enterprise` | Private |
| `muxon-enterprise-build` | Private |
| `muxon-design` | Private |
| `openspec` | Private |

Build helpers:

- OSS: `muxon-core-build`
- Enterprise/appliance: `muxon-enterprise-build`

`muxon-web` and `muxon-docs` are common packages (their own repos, not owned by Core or Enterprise).

## Required public PR checks (`muxon-core`)

Protect `muxon-core` `main` so Core-affecting merges require:

- **`core-build`**
- **`enterprise-compatibility`** (opaque status from `muxon-enterprise` CI)
- Public aggregate checks as configured in the Core repo

Web/docs-only PRs live in their own repos and do not require Enterprise compatibility by default.

## Image builds

```bash
cd muxon-core-build && ./scripts/publish-images-to-ghcr.sh oss
```

Enterprise images/appliances use `muxon-enterprise-build`.

## Appliance builds

```bash
# from muxon-enterprise-build repo workflows
gh workflow run appliance-build.yml -f variant=enterprise -f version=0.1.0
```

## Integration tests

Private/self-hosted Proxmox workflows live with Enterprise (`muxon-enterprise` / related private CI), not in public repos.
