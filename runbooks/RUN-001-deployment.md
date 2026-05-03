# RUN-001: Deployment

Deploying or updating any service. Same procedure every time.

---

## Prerequisites

- Service code committed and pushed to its GitHub repo
- `/srv/<service>/` exists on the target host (create if first deployment — see step 6)
- Host-specific override file exists at `Admin/compose/<host>/<service>/docker-compose.override.yml`
- Alexandria registry is reachable at `irina:5000` from the target host

---

## What Lives on the Host

Hosts never hold service source code. They hold only the two compose manifests needed to run `docker compose` operations (up, down, logs, restart, config):

```
/srv/.admin/<service>/
  docker-compose.yml              # cached from the service repo
  docker-compose.override.yml     # from Admin repo
```

These are manifests, not source. The container image, the Dockerfile, the application code — none of it lives on the host. Just the two files that tell `docker compose` what to run.

---

## Steps

### 1. Push code to GitHub

```bash
git push origin <branch>
```

GitHub is the source of truth. Nothing deploys directly from a developer's machine.

### 2. SSH to the target host

```bash
ssh aristotle9@<host>
```

### 3. Build the image

Clone the service repo to a throwaway location, build, then keep only what's needed:

```bash
cd /tmp
git clone https://github.com/<org>/<service>.git
cd <service>
TAG=$(git rev-parse --short HEAD)
docker build \
  --build-arg PIP_INDEX_URL=http://irina:3141/root/pypi/+simple/ \
  --build-arg NPM_CONFIG_REGISTRY=http://irina:4873 \
  -t irina:5000/<service>:$TAG .
```

### 4. Push the image to Alexandria

```bash
docker push irina:5000/<service>:$TAG
docker tag irina:5000/<service>:$TAG irina:5000/<service>:latest
docker push irina:5000/<service>:latest
```

### 5. Cache the compose file on the host

```bash
sudo mkdir -p /srv/.admin/<service>
sudo cp /tmp/<service>/docker-compose.yml /srv/.admin/<service>/
```

The override file at `/srv/.admin/<service>/docker-compose.override.yml` should already be in place — it was placed there when the host was set up (or via `scp` from the Admin repo as part of introducing the service to this host).

### 6. First-time setup (if needed)

If `/srv/<service>/` doesn't exist on this host:

```bash
sudo mkdir -p /srv/<service>
sudo chmod 777 /srv/<service>
```

### 7. Deploy

```bash
cd /srv/.admin/<service>
docker compose pull
docker compose up -d
```

Compose auto-detects both `docker-compose.yml` and `docker-compose.override.yml` in the current directory and merges them. No `-f` flags needed.

### 8. Clean up the build checkout

```bash
rm -rf /tmp/<service>
```

The persistent manifests at `/srv/.admin/<service>/` stay. Future `docker compose logs`, `restart`, `down`, `config` ops run from that directory.

### 9. Verify

```bash
docker ps --filter name=<service>
curl -f http://localhost:<port>/health
```

---

## Rollback

Roll back by editing the **override file** to pin a specific image SHA, then redeploying. This avoids mutating `:latest` on the registry (which would affect every host, not just this one).

```bash
cd /srv/.admin/<service>
```

Edit `docker-compose.override.yml`:
```yaml
services:
  <service>:
    image: irina:5000/<service>:<previous-sha>   # pinned, not :latest
    # ... rest of override unchanged
```

Redeploy:
```bash
docker compose pull
docker compose up -d
```

Every previously deployed image stays in the registry indefinitely, so any SHA is always rollback-able. When you're ready to return to the current line, change the tag back to `:latest` (or a newer SHA) and redeploy.

---

## Subsequent Compose Operations

Because both manifests live at `/srv/.admin/<service>/`, ongoing operations work from that directory without any checkout:

```bash
cd /srv/.admin/<service>
docker compose logs -f
docker compose restart
docker compose down
docker compose config     # validate merged output
```

---

## What This Procedure Does Not Include

- **Database migrations** — service-specific, documented in each service's own repo
- **Data migration between hosts** — see per-service migration docs; no generic procedure covers this
- **Host setup** — see `RUN-002-host-setup.md`
