# Local auto-deploy from GHCR

Operators can poll GitHub Container Registry for new `main` images and redeploy without building locally.

## Script

`muxon-build/scripts/watch-ghcr-deploy.sh`

| Env | Default | Meaning |
|-----|---------|---------|
| `GHCR_REGISTRY` | (required) | Image without tag, e.g. `ghcr.io/OWNER/scal/core-services` |
| `IMAGE_TAG` | `main` | Tag to watch |
| `COMPOSE_FILE` | `muxon-core/compose/muxon.yml` | Compose stack |
| `POLL_INTERVAL` | `300` | Seconds between polls (`0` = one-shot) |
| `HEALTH_URL` | `http://localhost:8080/actuator/health` | Post-deploy gate |
| `STATE_FILE` | `~/.infron/last-image-digest` | Last successful digest |

Example one-shot:

```bash
export GHCR_REGISTRY=ghcr.io/scal/scal/core-services
export IMAGE_TAG=main
export POLL_INTERVAL=0
./muxon-build/scripts/watch-ghcr-deploy.sh
```

Authenticate Docker to GHCR first (`docker login ghcr.io` or `gh auth token`).

## systemd timer example

Unit files live under `muxon-build/deploy/systemd/`:

```bash
sudo cp muxon-build/deploy/systemd/infron-ghcr-watch.* /etc/systemd/system/
# Edit Environment= in the service file for your registry/owner
sudo systemctl daemon-reload
sudo systemctl enable --now infron-ghcr-watch.timer
```
