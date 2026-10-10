# Go Backend — Docker Containerization and CI/CD Migration

**Project:** `nohj-luap/johnpaulabecia-official-backend`  
**Period:** October 9–10, 2026  
**Production endpoint:** https://api.sannycom.com/  
**Final deployed commit:** `07b1f1651c3a12303b64df180646f63e6e047bd9`

> **Evidence policy:** This records actual configuration, commands, outputs, and confirmed results from the available conversation. Where exact earlier installation or troubleshooting commands are unavailable, that gap is stated rather than filled with invented historical steps. The prior systemd CI/CD documentation describes the *starting state*, not this final Docker deployment.

## 1. Initial production state and migration goal

The existing AWS EC2 Ubuntu server ran a native Go backend under `johnpaulabecia-backend.service` and host-level Nginx. The Go service listened on `8080`; Cloudflare and existing origin certificates supported the public HTTPS API. Production database services were hosted on Supabase, not EC2.

The pre-migration verification confirmed both host services active, certificates at `/etc/nginx/ssl/api.sannycom.com.pem` and `/etc/nginx/ssl/api.sannycom.com.key`, about **2.8 GiB disk available**, **534 MiB RAM available**, and **no swap**. The objective was to move both the backend and Nginx into Docker Compose without prematurely disrupting production.

### Final architecture

```text
git push main
    │
    ▼
GitHub Actions Continuous Integration
    │ success on push to main
    ▼
GitHub Actions Continuous Deployment
    ├─ Checkout CI-verified SHA
    ├─ Docker Buildx → GHCR image tagged with commit SHA
    ├─ Checkout private DevOps repository
    ├─ SCP Compose + Nginx files to EC2
    └─ SSH: validate → pull → deploy → health → HTTPS check
                                   │ failure
                                   ▼
                          restore previous image
                          verify image + health + API

Client → Cloudflare HTTPS → EC2 Docker Nginx (80/443)
                                 │ Compose network
                                 ▼
                            Go backend (8080)
                                 │
                                 ▼
                         Supabase PostgreSQL
```

## 2. Docker installation and preparation

**Verified versions on EC2:**

| Component | Version |
|---|---|
| Docker Engine | `29.9.0` |
| Docker Compose | `v5.6.0` |
| Docker Buildx | `v0.38.0` |

Docker group access worked. Host Nginx and the host Go service remained active during installation. **The earlier numbered conversation has now been consulted for the Docker installation commands.** The commands below were part of the recorded instructions; individual package-manager output lines are not all retained.

Useful version and service verification commands (reference commands, not asserted as the exact historical sequence):

```bash
docker --version
docker compose version
docker buildx version
systemctl is-active docker containerd
```


### 2.1 — Update apt and install prerequisites (recorded instruction)

On the **EC2 Ubuntu** shell:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
dpkg -l ca-certificates curl | grep '^ii'
```

The final command checked that both prerequisite packages were installed before configuring Docker's apt repository.

### 2.2 — Add Docker's official Ubuntu repository (recorded instruction)

The documented approach used `/etc/apt/keyrings/docker.asc` and the Deb822 `/etc/apt/sources.list.d/docker.sources` file:

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
apt-cache policy docker-ce
```

**Evidence distinction:** The source conversation confirms the GPG-key path, repository URL, codename selection, `Signed-By`, `apt update`, and `apt-cache policy` steps. The block above is a faithful *reconstruction* of the recorded procedure, not a verbatim terminal transcript.

### 2.3 — Install Docker Engine, Compose, and Buildx (recorded instruction)

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

sudo systemctl enable --now docker

docker --version
docker compose version
docker buildx version
sudo systemctl is-active docker
```

**Observed successful versions:**

```text
Docker Engine: 29.9.0
Docker Compose: v5.6.0
Docker Buildx: v0.38.0
Docker daemon: active
```

The disk-free observation after installation was approximately **2.5 GiB**.

### 2.4 — Allow the `ubuntu` user to access Docker (recorded instruction)

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

The subsequent Docker-access verification printed:

```text
Docker access OK
```

Docker group membership grants broad privileges over the host; it is not equivalent to a limited application role.

### 2.5 — Verify that production was not interrupted

The recorded result at the end of Docker installation was that the existing host Nginx and host systemd backend were still active. This was deliberate: **install Docker first; switch traffic later**.

## 3. Repository and production paths

| Item | Actual path |
|---|---|
| Backend repository | `nohj-luap/johnpaulabecia-official-backend` |
| DevOps repository | `nohj-luap/johnpaulabecia-official-devops` |
| EC2 DevOps directory | `/Applications/johnpaulabecia-apps/johnpaulabecia-official-devops` |
| Production Compose | `compose/docker-compose.production.yml` |
| Production Nginx config | `nginx/default.production.conf` |
| Production environment | `/Applications/johnpaulabecia-apps/johnpaulabecia-official-backend/.env.production` |
| GHCR image | `ghcr.io/nohj-luap/johnpaulabecia-official-backend:<commit-sha>` |

The `.env.production` file stays on EC2; CD transfers only Compose and Nginx configuration. EC2's `ubuntu` account was authenticated to GHCR so it could pull the private image.


## 3A. Step 3 — Build and locally test the Go container

### 3A.1 — Confirm the backend structure before writing the Dockerfile

The user confirmed these actual application facts:

```text
Go version: 1.27.1
Application entrypoint: cmd/server
Migration entrypoint: cmd/migrate
Environment configuration: os.Getenv
Initial Dockerfile: absent
Initial .dockerignore: absent
```

The migration kept `cmd/migrate` **out of the production image entrypoint**. Production database migration continued to be managed separately.

### 3A.2 — Create a multi-stage Dockerfile

The recorded design used:

- `golang:1.27.1-alpine` as the builder image.
- `CGO_ENABLED=0`, Linux AMD64 target.
- A separate `alpine:3.22` runtime.
- CA certificates and time-zone data in the runtime.
- A non-root application account.
- Port `8080`, with `/app/backend` as the container command.

**Representative Dockerfile implementing that recorded design** (exact original Dockerfile bytes were not preserved in the retrieved excerpts):

```dockerfile
FROM golang:1.27.1-alpine AS builder

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -o /app/backend ./cmd/server

FROM alpine:3.22

RUN apk add --no-cache ca-certificates tzdata \
    && addgroup -S app \
    && adduser -S -G app app

WORKDIR /app
COPY --from=builder /app/backend /app/backend

USER app
EXPOSE 8080
CMD ["/app/backend"]
```

The `.dockerignore` excluded secret-bearing environment files and other irrelevant development content. **Do not copy `.env.production` into the image.**

### 3A.3 — Actual local Docker image build

The following command and successful build output are preserved in a pasted terminal attachment:

```bash
docker buildx build \
  --platform linux/amd64 \
  --load \
  -t johnpaulabecia-backend:local \
  .
```

**Observed result:**

```text
[+] Building 32.8s (16/16) FINISHED
...
load build definition from Dockerfile
load metadata for docker.io/library/golang:1.27.1-alpine
load metadata for docker.io/library/alpine:3.22
load .dockerignore
```

The build succeeded; the multi-stage Dockerfile and ignore file were read by Buildx.

### 3A.4 — Local runtime and database integration test

The container was tested on the local Docker network:

```text
compose_johnpaulabecia-development-network
```

The application successfully connected to PostgreSQL and logged that it was listening on `:8080` **inside the container**.

A host-side request to **port 8081** succeeded:

```bash
curl -i http://127.0.0.1:8081/
```

**Observed:** HTTP 200, response body:

```text
John Paul Abecia Portfolio Official API
```

The port distinction matters: the local host test used **8081**, while the application listened on **8080** inside the container. Production later used private container port 8080 behind Nginx rather than exposing 8081.

### 3A.5 — Production reverse-proxy preparation

The existing Cloudflare Origin Certificate files were already on EC2:

```text
/etc/nginx/ssl/api.sannycom.com.pem
/etc/nginx/ssl/api.sannycom.com.key
```

The Docker migration **reused** them through the Nginx read-only mount instead of issuing new certificates.

The production Nginx configuration moved into the DevOps repository at:

```text
nginx/default.production.conf
```

It routed requests to the Compose backend by Docker DNS/service name (`backend:8080`), replacing the old host-based `127.0.0.1:8080` upstream.

**Exact final Nginx configuration text was not retained** in the available source; consult the private DevOps repository rather than treating a fabricated server block as historical.

## 4. Docker Compose production configuration

This reproduces the **recorded final service design**, including the later health-check change:

```yaml
name: johnpaulabecia-production

services:
  backend:
    image: ghcr.io/nohj-luap/johnpaulabecia-official-backend:${IMAGE_TAG:?IMAGE_TAG is required}
    restart: unless-stopped
    env_file:
      - /Applications/johnpaulabecia-apps/johnpaulabecia-official-backend/.env.production
    expose:
      - "8080"
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://127.0.0.1:8080/"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
    networks:
      - backend-network
    security_opt:
      - "no-new-privileges:true"

  nginx:
    image: nginx:stable-alpine
    restart: unless-stopped
    depends_on:
      - backend
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ../nginx/default.production.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/nginx/ssl:/etc/nginx/ssl:ro
    networks:
      - backend-network
    security_opt:
      - "no-new-privileges:true"

networks:
  backend-network:
    driver: bridge
```

Key choices: backend is only exposed inside the Compose network; containerized Nginx owns host ports 80/443; existing certificates are mounted read-only; both containers restart unless stopped. The `wget` health command was tested inside the backend image.

### Compose validation

**Recorded successful local validation:**

```bash
IMAGE_TAG=5221a3a24db37a09ac27e1bfc98786a217f7a9fd \
  docker compose -f compose/docker-compose.production.yml \
  config --no-env-resolution --quiet
```

**The CD workflow also validates the transferred files on EC2:**

```bash
export IMAGE_TAG="$DEPLOY_SHA"
docker compose \
  -f /Applications/johnpaulabecia-apps/johnpaulabecia-official-devops/compose/docker-compose.production.yml \
  config --quiet
```

## 5. Migration cutover: host services to Docker

The verified sequence was:

1. Check host Go service, Nginx, certificates, disk, and memory.
2. Install and verify Docker while the existing host services remain active.
3. Prepare backend image, private GHCR access, production Compose, and containerized Nginx.
4. Start and verify the Compose-based backend and reverse proxy.
5. Stop and disable the former host `johnpaulabecia-backend` and host Nginx services after Docker replacements are operational.
6. Enable Docker/containerd at boot; use `restart: unless-stopped` for both containers.
7. Confirm the Cloudflare public API still responds with HTTP 200.

**The exact intermediate cutover commands and error messages are not fully preserved** in the accessible conversation. Do not invent a port-conflict incident or assert a particular stop/start command was typed.

**First verified deployed image before health/rollback changes:**

```text
ghcr.io/nohj-luap/johnpaulabecia-official-backend:5221a3a24db37a09ac27e1bfc98786a217f7a9fd
```

The backend was subsequently redeployed independently without restarting Nginx.

## 6. CI-gated CD workflow

**Backend file:** `.github/workflows/cd.yml`

**Trigger and guard:**

```yaml
name: Continuous Deployment

on:
  workflow_run:
    workflows:
      - Continuous Integration
    types:
      - completed
    branches:
      - main

permissions:
  contents: read
  packages: write

concurrency:
  group: production-backend-deployment
  cancel-in-progress: false

jobs:
  deploy:
    if: >-
      ${{
        github.event.workflow_run.conclusion == 'success' &&
        github.event.workflow_run.event == 'push' &&
        github.event.workflow_run.head_branch == 'main' &&
        github.event.workflow_run.head_repository.full_name == github.repository
      }}
    runs-on: ubuntu-latest
```

**Nine recorded workflow steps:**

| # | Action | Purpose |
|---|---|---|
| 1 | `actions/checkout@v4` | Checkout exact `${{ github.event.workflow_run.head_sha }}` |
| 2 | `docker/setup-buildx-action@v3` | Buildx |
| 3 | `docker/login-action@v3` | GHCR authentication with `GITHUB_TOKEN` |
| 4 | `docker/build-push-action@v6` | Build `linux/amd64` and push commit-SHA image |
| 5 | `actions/checkout@v4` | Private DevOps repository via `DEVOPS_DEPLOY_KEY` |
| 6 | Shell copy | Stage production Compose and Nginx config |
| 7 | `appleboy/scp-action@v1` | Transfer files to EC2 |
| 8 | `appleboy/ssh-action@v1.0.3` | Verify transferred files and Compose config |
| 9 | `appleboy/ssh-action@v1.0.3` | Deploy, verify, and automatically roll back on error |

**Relevant GitHub secrets:** `EC2_HOST`, `EC2_SSH_KEY`, `DEVOPS_DEPLOY_KEY`. `GITHUB_TOKEN` is supplied by Actions. The SSH username is `ubuntu`. Secret values are intentionally excluded.

**Recorded SCP settings:**

```yaml
source: "deployment/compose/docker-compose.production.yml,deployment/nginx/default.production.conf"
target: "/Applications/johnpaulabecia-apps/johnpaulabecia-official-devops"
strip_components: 1
```

**Deployment image and verification commands (from final CD):**

```bash
cd /Applications/johnpaulabecia-apps/johnpaulabecia-official-devops

COMPOSE_FILE="compose/docker-compose.production.yml"
CONTAINER="johnpaulabecia-production-backend-1"
IMAGE_REPO="ghcr.io/nohj-luap/johnpaulabecia-official-backend"

PREVIOUS_IMAGE=$(docker inspect --format '{{.Config.Image}}' "$CONTAINER")
PREVIOUS_TAG="${PREVIOUS_IMAGE##*:}"

docker image inspect "$PREVIOUS_IMAGE" > /dev/null
export IMAGE_TAG="$DEPLOY_SHA"

# Pull before touching the currently running backend:
docker compose -f "$COMPOSE_FILE" pull backend

# Enable rollback trap, then replace backend only:
trap rollback ERR
docker compose -f "$COMPOSE_FILE" up -d --no-deps backend

RUNNING_IMAGE=$(docker inspect --format '{{.Config.Image}}' "$CONTAINER")
if [ "$RUNNING_IMAGE" != "$IMAGE_REPO:$DEPLOY_SHA" ]; then
  false
fi

# The actual script also waits for Docker Health.Status == healthy.
curl --fail --silent --show-error \
  --retry 5 --retry-delay 3 --max-time 10 \
  https://api.sannycom.com/ > /dev/null
```

**Important:** The snippet above illustrates the main operations, not a standalone deploy script: it references the separately defined `rollback` function and `DEPLOY_SHA`. The authoritative complete executable script is the repository's `.github/workflows/cd.yml`.

## 7. Health check and rollback implementation

**DevOps commit:** `8cdaa93` added the backend health check.

**Recorded verification of committed DevOps configuration:**

```bash
git show origin/main:compose/docker-compose.production.yml | grep -A 5 'healthcheck:'
```

**Actual output:**

```yaml
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://127.0.0.1:8080/"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
```

### Automatic rollback

The CD Bash script uses `set -Eeuo pipefail` and installs `trap rollback ERR` after successfully pulling the new image. It checks the new image reference, polls container health up to 30 times (5 seconds apart), and checks public HTTPS.

On deployment/verification failure it:

1. Disables the error trap to avoid recursion.
2. Restores the previous image tag.
3. Recreates **only** the backend with the previous image.
4. Verifies the restored image reference.
5. Waits for restored Docker health.
6. Rechecks the public API.
7. Exits nonzero even if restoration succeeds, correctly marking the failed release.

**Core restore command:**

```bash
export IMAGE_TAG="$PREVIOUS_TAG"
docker compose -f "$COMPOSE_FILE" up -d --no-deps --force-recreate backend
```

**Rollback source actually verified on EC2:**

```bash
docker image inspect \
  "$(docker inspect --format '{{.Config.Image}}' johnpaulabecia-production-backend-1)" \
  --format 'Rollback source available: {{.RepoTags}}'
```

**Actual output:**

```text
Rollback source available: [ghcr.io/nohj-luap/johnpaulabecia-official-backend:5221a3a24db37a09ac27e1bfc98786a217f7a9fd]
```

**Actual local Bash error-trap smoke test:**

```bash
bash -c '
set -Eeuo pipefail
rollback() {
  echo "Rollback triggered successfully"
  exit 1
}
trap rollback ERR
false
'
```

**Actual output:**

```text
Rollback triggered successfully
```

This **did not** exercise a real Docker rollback. The recovery path is configured but not end-to-end failure-tested.

## 8. Workflow validation

**Actual command:**

```bash
$(go env GOPATH)/bin/actionlint
```

**Actual result:** No output (passed).

**Actual embedded Bash validation:**

```bash
python3 - <<'PY'
from pathlib import Path
import subprocess
import yaml

workflow = yaml.safe_load(
    Path(".github/workflows/cd.yml").read_text()
)
script = workflow["jobs"]["deploy"]["steps"][-1]["with"]["script"]

result = subprocess.run(
    ["bash", "-n"],
    input=script,
    text=True,
    capture_output=True,
)
if result.returncode:
    print(result.stderr)
    raise SystemExit(result.returncode)

print("Deployment Bash syntax valid")
PY
```

**Actual output:**

```text
Deployment Bash syntax valid
```

## 9. Commits, push, and Actions verification

Recorded revisions:

- `8cdaa93` — DevOps health-check configuration pushed.
- `e4a11dd` — Initial local commit for automatic rollback and health verification.
- `07b1f16` — Final pushed backend `main` revision including rollback-health verification.

**Actual final push command:**

```bash
git push origin main
```

**Actual output:**

```text
To https://github.com/nohj-luap/johnpaulabecia-official-backend.git
   5221a3a..07b1f16  main -> main
```

**User-reported verification:** both **Continuous Integration** and **Continuous Deployment** were green.

## 10. Final EC2 and HTTPS verification

### Container image, running state, health

**Actual command:**

```bash
docker inspect \
  --format 'Image: {{.Config.Image}}
Status: {{.State.Status}}
Health: {{.State.Health.Status}}' \
  johnpaulabecia-production-backend-1
```

**Actual output:**

```text
Image: ghcr.io/nohj-luap/johnpaulabecia-official-backend:07b1f1651c3a12303b64df180646f63e6e047bd9
Status: running
Health: healthy
```

### Public API

**Actual command (local Ubuntu):**

```bash
curl -i https://api.sannycom.com/
```

**Relevant actual output:**

```http
HTTP/2 200
date: Sat, 10 Oct 2026 04:40:20 GMT
content-type: text/plain; charset=utf-8
content-length: 47
server: cloudflare
cf-cache-status: DYNAMIC

John Paul Abecia Portfolio Official Backend API
```

## 11. Failures, risk resolutions, and boundaries

| Problem or risk | Actual resolution / evidence |
|---|---|
| Existing host production services | Preserved while installing Docker; disabled only after successful container migration |
| Docker resource constraints | Disk/memory checked before installation |
| Private image distribution | EC2 registry authentication configured; GHCR image pulled successfully |
| Nginx restarts during backend deployment | `up -d --no-deps backend`; Nginx remained up |
| Wrong backend version | SHA-tagged image plus exact image-reference check |
| Container starts but app fails | Docker `wget` health check and polling |
| Failed release | Previous-image rollback trap and restoration checks configured |
| Syntax mistakes | `actionlint` and `bash -n` passed |
| CI/CD sequencing | CD now depends on successful CI `workflow_run` |
| Public route broken | HTTPS `curl` returned HTTP/2 200 |

**Do not infer unrecorded incidents:** the available conversation does not include the full exact command/error transcript for Docker installation and initial cutover. This table distinguishes preventive fixes from demonstrated failures.

**Remaining limitations:** rollback has not been tested through an intentionally failed deployment; rollback does not restore database migrations or older Compose/Nginx files; a single backend container can have brief redeploy downtime; a public root health check does not test every API; GHCR credentials must remain valid. The earlier pre-Docker deployment temporarily opened SSH ingress to `0.0.0.0/0`; the available record does not establish whether that rule was later restricted.

## 12. Operational reference commands (not necessarily historically executed)

```bash
# On EC2
docker ps
docker logs --tail 100 johnpaulabecia-production-backend-1
docker inspect --format '{{.Config.Image}}' johnpaulabecia-production-backend-1
docker inspect --format '{{.State.Status}} / {{.State.Health.Status}}' johnpaulabecia-production-backend-1
systemctl is-enabled docker containerd

# From an external client
curl -i https://api.sannycom.com/
```

## 13. Documentation provenance

The previous uploaded `GO_BACKEND_CI_CD_SETUP_AND_VERIFICATION.md` and `ci-cd-pipeline.pdf` document the **earlier systemd-based CI/CD setup**, including previous GitHub SSH and Go build troubleshooting. This document instead captures the **subsequent October 10 Docker migration** and its verified final results from the conversation. The complete final CD source of truth remains `.github/workflows/cd.yml` in the backend repository; Compose and Nginx source of truth remain in the private DevOps repository.

**Final verified outcome:** Both CI and CD passed; EC2 ran image `07b1f1651c3a12303b64df180646f63e6e047bd9` with Docker status `running` and health `healthy`; `https://api.sannycom.com/` returned `HTTP/2 200` and the expected application body.


## 14. Exact final workflow: `.github/workflows/cd.yml`

The following is the **full final workflow supplied in the conversation** after the rollback-health verification was added. Unlike the partial examples earlier in this document, this is intended as a complete reference for the production workflow at the end of Step 6.35.

```yaml
name: Continuous Deployment

on:
  workflow_run:
    workflows:
      - Continuous Integration
    types:
      - completed
    branches:
      - main

permissions:
  contents: read
  packages: write

concurrency:
  group: production-backend-deployment
  cancel-in-progress: false

jobs:
  deploy:
    if: >-
      ${{
        github.event.workflow_run.conclusion == 'success' &&
        github.event.workflow_run.event == 'push' &&
        github.event.workflow_run.head_branch == 'main' &&
        github.event.workflow_run.head_repository.full_name == github.repository
      }}

    runs-on: ubuntu-latest

    steps:
      # 1. Checkout the exact commit verified by CI.
      - name: Checkout verified backend commit
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
          persist-credentials: false

      # 2. Configure Docker Buildx.
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # 3. Authenticate to GitHub Container Registry.
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # 4. Build and publish the production backend image.
      - name: Build and push backend image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64
          push: true
          tags: |
            ghcr.io/nohj-luap/johnpaulabecia-official-backend:${{ github.event.workflow_run.head_sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      # 5. Retrieve the private DevOps repository.
      - name: Checkout DevOps repository
        uses: actions/checkout@v4
        with:
          repository: nohj-luap/johnpaulabecia-official-devops
          ssh-key: ${{ secrets.DEVOPS_DEPLOY_KEY }}
          path: devops
          persist-credentials: false

      # 6. Prepare production deployment files.
      - name: Prepare deployment files
        run: |
          set -eu

          mkdir -p deployment/compose deployment/nginx

          cp devops/compose/docker-compose.production.yml \
            deployment/compose/docker-compose.production.yml

          cp devops/nginx/default.production.conf \
            deployment/nginx/default.production.conf

      # 7. Transfer deployment configuration to EC2.
      - name: Transfer configuration to EC2
        uses: appleboy/scp-action@v1
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "deployment/compose/docker-compose.production.yml,deployment/nginx/default.production.conf"
          target: "/Applications/johnpaulabecia-apps/johnpaulabecia-official-devops"
          strip_components: 1

      # 8. Verify transferred production configuration.
      - name: Verify EC2 deployment files
        uses: appleboy/ssh-action@v1.0.3
        env:
          DEPLOY_SHA: ${{ github.event.workflow_run.head_sha }}
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          envs: DEPLOY_SHA
          script: |
            set -eu

            DEPLOY_DIR="/Applications/johnpaulabecia-apps/johnpaulabecia-official-devops"

            test -f "$DEPLOY_DIR/compose/docker-compose.production.yml"
            test -f "$DEPLOY_DIR/nginx/default.production.conf"

            export IMAGE_TAG="$DEPLOY_SHA"

            docker compose \
              -f "$DEPLOY_DIR/compose/docker-compose.production.yml" \
              config --quiet

            echo "Production configuration verified."
            echo "Backend image: ghcr.io/nohj-luap/johnpaulabecia-official-backend:$DEPLOY_SHA"

      # 9. Deploy the verified backend image with automatic rollback.
      - name: Deploy Docker Compose to EC2
        uses: appleboy/ssh-action@v1.0.3
        env:
          DEPLOY_SHA: ${{ github.event.workflow_run.head_sha }}
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          envs: DEPLOY_SHA
          command_timeout: 10m
          script: |
            bash -euo pipefail <<'DEPLOY_SCRIPT'
            set -Eeuo pipefail

            cd /Applications/johnpaulabecia-apps/johnpaulabecia-official-devops

            COMPOSE_FILE="compose/docker-compose.production.yml"
            CONTAINER="johnpaulabecia-production-backend-1"
            IMAGE_REPO="ghcr.io/nohj-luap/johnpaulabecia-official-backend"

            # Capture the currently deployed image.
            PREVIOUS_IMAGE=$(docker inspect \
              --format '{{.Config.Image}}' \
              "$CONTAINER")

            PREVIOUS_TAG="${PREVIOUS_IMAGE##*:}"

            echo "Previous image: $PREVIOUS_IMAGE"
            echo "New image: $IMAGE_REPO:$DEPLOY_SHA"

            # Ensure the previous image is available for rollback.
            docker image inspect "$PREVIOUS_IMAGE" > /dev/null

            # Verify that the previous image belongs to this repository.
            if [[ "$PREVIOUS_IMAGE" != "$IMAGE_REPO:"* ]]; then
              echo "ERROR: Previous image repository does not match."
              exit 1
            fi

            export IMAGE_TAG="$DEPLOY_SHA"

            # Verify Docker container health.
            wait_for_health() {
              local container="$1"
              local label="$2"
              local health=""

              for attempt in $(seq 1 30); do
                health=$(docker inspect \
                  --format '{{if .State.Health}}{{.State.Health.Status}}{{else}}missing{{end}}' \
                  "$container") || return 1

                echo "$label health check $attempt/30: $health"

                case "$health" in
                  healthy)
                    return 0
                    ;;
                  unhealthy|missing)
                    return 1
                    ;;
                esac

                sleep 5
              done

              return 1
            }

            # Restore the previous backend image after deployment failure.
            rollback() {
              local exit_code=$?

              trap - ERR
              set +e

              echo "ERROR: Deployment failed with exit code $exit_code."
              echo "Starting automatic rollback..."

              export IMAGE_TAG="$PREVIOUS_TAG"

              docker compose \
                -f "$COMPOSE_FILE" \
                up -d --no-deps --force-recreate backend

              local rollback_code=$?

              if [ "$rollback_code" -eq 0 ]; then
                echo "Previous backend image restored."
                echo "Restored image: $PREVIOUS_IMAGE"

                # Verify the restored image.
                RESTORED_IMAGE=$(docker inspect \
                  --format '{{.Config.Image}}' \
                  "$CONTAINER")

                if [ "$RESTORED_IMAGE" != "$PREVIOUS_IMAGE" ]; then
                  echo "CRITICAL: Restored image does not match previous image."
                  exit 1
                fi

                # Verify the restored backend becomes healthy.
                if wait_for_health "$CONTAINER" "Rollback"; then
                  echo "Rollback health verification successful."

                  # Verify public API after rollback.
                  if curl --fail --silent --show-error \
                    --retry 5 \
                    --retry-delay 3 \
                    --max-time 10 \
                    https://api.sannycom.com/ > /dev/null; then

                    echo "Rollback successful. Production API is responding."
                  else
                    echo "CRITICAL: Public API verification failed after rollback."
                  fi
                else
                  echo "CRITICAL: Restored backend did not become healthy."
                fi
              else
                echo "CRITICAL: Automatic rollback failed."
                echo "Manual intervention is required."
              fi

              # Deployment remains failed even when rollback succeeds.
              exit 1
            }

            # Pull the new image before modifying production.
            echo "Pulling new backend image..."

            docker compose \
              -f "$COMPOSE_FILE" \
              pull backend

            # Enable rollback for deployment failures.
            trap rollback ERR

            echo "Deploying new backend..."

            docker compose \
              -f "$COMPOSE_FILE" \
              up -d --no-deps backend

            # Verify the deployed image matches the CI commit.
            RUNNING_IMAGE=$(docker inspect \
              --format '{{.Config.Image}}' \
              "$CONTAINER")

            if [ "$RUNNING_IMAGE" != "$IMAGE_REPO:$DEPLOY_SHA" ]; then
              echo "ERROR: Deployed image does not match expected SHA."
              false
            fi

            # Wait for Docker to report the backend as healthy.
            echo "Waiting for backend health check..."

            if ! wait_for_health "$CONTAINER" "Deployment"; then
              echo "ERROR: Backend did not become healthy."
              false
            fi

            # Verify the public HTTPS API.
            echo "Verifying production API..."

            curl --fail --silent --show-error \
              --retry 5 \
              --retry-delay 3 \
              --max-time 10 \
              https://api.sannycom.com/ > /dev/null

            # Deployment succeeded. Disable automatic rollback.
            trap - ERR

            echo "Production deployment successful."
            echo "Deployed image: $RUNNING_IMAGE"

            docker compose -f "$COMPOSE_FILE" ps

            DEPLOY_SCRIPT
```

## 15. Step-by-step verification transcript: Steps 6.24–6.35

This section follows the **actual final numbered interactions**. It distinguishes tests of the script from tests of the live service.

| Step | Action and result |
|---|---|
| **6.24** | Python/PyYAML extracted the final SSH `script` from `.github/workflows/cd.yml` and passed it to `bash -n`; output `Deployment Bash syntax valid`. |
| **6.25** | Ran `bash -c` with `set -Eeuo pipefail`, `trap rollback ERR`, and intentional `false`; output `Rollback triggered successfully`. |
| **6.26** | EC2 `docker image inspect "$(docker inspect ...)"` confirmed previous image `5221a3a24db37a09ac27e1bfc98786a217f7a9fd` existed locally. |
| **6.27** | Local DevOps `git show origin/main:compose/docker-compose.production.yml \\| grep -A 5 'healthcheck:'` showed the five-line `wget` health-check configuration. |
| **6.28** | Revised `rollback()` to verify that the restored backend image and Docker health status were correct; final workflow included public HTTPS recheck. |
| **6.29** | Ran `$(go env GOPATH)/bin/actionlint`; **no output**. |
| **6.30** | Repeated Python/PyYAML extraction and `bash -n` for the finalized workflow; output `Deployment Bash syntax valid`. |
| **6.31** | Committed the finalized `.github/workflows/cd.yml` locally with message `ci: verify backend health after automatic rollback`. |
| **6.32** | `git push origin main` succeeded: `5221a3a..07b1f16 main -> main`. |
| **6.33** | User confirmed **both CI and CD green** in GitHub Actions. |
| **6.34** | EC2 `docker inspect` reported exact image `07b1f1651c3a12303b64df180646f63e6e047bd9`, `Status: running`, `Health: healthy`. |
| **6.35** | Local `curl -i https://api.sannycom.com/` returned `HTTP/2 200` and `John Paul Abecia Portfolio Official Backend API`. |

### 6.24 and 6.30 — Exact syntax verification

```bash
python3 - <<'PY'
from pathlib import Path
import subprocess
import yaml

workflow = yaml.safe_load(
    Path(".github/workflows/cd.yml").read_text()
)

script = workflow["jobs"]["deploy"]["steps"][-1]["with"]["script"]

result = subprocess.run(
    ["bash", "-n"],
    input=script,
    text=True,
    capture_output=True,
)

if result.returncode:
    print(result.stderr)
    raise SystemExit(result.returncode)

print("Deployment Bash syntax valid")
PY
```

**Observed both times:**

```text
Deployment Bash syntax valid
```

### 6.25 — Exact error trap test

```bash
bash -c '
set -Eeuo pipefail

rollback() {
  echo "Rollback triggered successfully"
  exit 1
}

trap rollback ERR

false
'
```

**Observed:**

```text
Rollback triggered successfully
```

### 6.26 — Exact previous-image check on EC2

```bash
docker image inspect \
  "$(docker inspect --format '{{.Config.Image}}' johnpaulabecia-production-backend-1)" \
  --format 'Rollback source available: {{.RepoTags}}'
```

**Observed:**

```text
Rollback source available: [ghcr.io/nohj-luap/johnpaulabecia-official-backend:5221a3a24db37a09ac27e1bfc98786a217f7a9fd]
```

### 6.27 — Exact DevOps health-check check

```bash
git show origin/main:compose/docker-compose.production.yml | grep -A 5 'healthcheck:'
```

**Observed:**

```yaml
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://127.0.0.1:8080/"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
```

### 6.29 — GitHub Actions static validation

```bash
$(go env GOPATH)/bin/actionlint
```

**Observed:** no output.

### 6.31–6.33 — Commit, push, CI/CD

```bash
git add .github/workflows/cd.yml
git commit -m "ci: verify backend health after automatic rollback"
git push origin main
```

**Observed push:**

```text
Enumerating objects: 14, done.
Counting objects: 100% (14/14), done.
Delta compression using up to 12 threads
Compressing objects: 100% (8/8), done.
Writing objects: 100% (10/10), 2.64 KiB | 1.32 MiB/s, done.
Total 10 (delta 4), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (4/4), completed with 2 local objects.
To https://github.com/nohj-luap/johnpaulabecia-official-backend.git
   5221a3a..07b1f16  main -> main
```

**GitHub Actions:** CI green; CD green.

### 6.34 — Exact production container inspection

```bash
docker inspect \
  --format 'Image: {{.Config.Image}}
Status: {{.State.Status}}
Health: {{.State.Health.Status}}' \
  johnpaulabecia-production-backend-1
```

**Observed:**

```text
Image: ghcr.io/nohj-luap/johnpaulabecia-official-backend:07b1f1651c3a12303b64df180646f63e6e047bd9
Status: running
Health: healthy
```

### 6.35 — Exact public HTTPS verification

```bash
curl -i https://api.sannycom.com/
```

**Observed (relevant headers and complete body):**

```http
HTTP/2 200
date: Sat, 10 Oct 2026 04:40:20 GMT
content-type: text/plain; charset=utf-8
content-length: 47
server: cloudflare
nel: {"report_to":"cf-nel","success_fraction":0.0,"max_age":604800}
vary: Origin
cf-cache-status: DYNAMIC
alt-svc: h3=":443"; ma=86400

John Paul Abecia Portfolio Official Backend API
```

The original terminal placed the next shell prompt immediately after the body because the response did not include a trailing newline.

## 16. Coverage audit — What is complete and what still requires original logs

| Phase | Evidence recovered | Completeness |
|---|---|---|
| EC2 preflight | Host services, TLS paths, disk, RAM | Substantive results known |
| Docker install | Actual apt prerequisite, repository, package, enable, group instructions; versions | Instructions recovered; complete apt output not recovered |
| Dockerfile | Go/Alpine versions, build target, runtime approach | Design known; example Dockerfile reconstructed |
| Local image build | Exact Buildx command, 32.8-second success | Confirmed |
| Local runtime test | Docker network, DB connectivity, `:8080` app, `:8081` host 200 | Confirmed |
| Production Compose | Final YAML, validation, health check | Confirmed |
| Nginx | Paths, container mount, network upstream | Final config file text not recovered |
| First GHCR deployment | Registry and initial SHA | Confirmed outcome; some manual commands missing |
| Host → Docker cutover | Service replacement and resulting operational state | Confirmed outcome; complete commands/logs missing |
| CI/CD | Complete final workflow and commit-SHA model | Confirmed |
| Rollback | Complete handler, Bash smoke test, previous image present | Configured, not failure-tested |
| Steps 6.24–6.35 | Exact commands and outputs above | Confirmed |

**No historical document can accurately invent the unretained commands or failures.** For an exact byte-for-byte account of those intermediate phases, the original shell history, Git commits, and GitHub Actions logs would be required.
