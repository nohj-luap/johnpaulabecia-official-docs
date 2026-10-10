# Deployment V1 Cleanup After CI/CD V2 Migration

**System:** John Paul Abecia Portfolio API (`api.sannycom.com`)  
**Host:** AWS EC2, Ubuntu 26.04.1 LTS, `ubuntu@ip-172-31-20-54`  
**Purpose:** Remove obsolete host-based deployment dependencies and Go caches after successful migration to the containerized CI/CD V2 deployment.  
**Result:** Production remained healthy; root filesystem usage decreased from approximately **65%** to **51%**.

> **Scope and evidence:** This runbook records the commands, observations, and outcomes available from the completed cleanup conversation. Some initial exploratory commands are reconstructed from the recorded results rather than preserved verbatim. They are labeled accordingly. This document does **not** claim a rollback drill was performed.

## 1. Deployment state and cleanup boundaries

CI/CD V2 deploys a prebuilt, SHA-tagged Go backend image from GitHub Container Registry (GHCR) using GitHub Actions and Docker Compose. Nginx also runs inside Docker. The EC2 host no longer needs its old host-installed Go compiler or Nginx service to serve production traffic.

- Compose project: `johnpaulabecia-production`
- Backend container: `johnpaulabecia-production-backend-1`
- Nginx container: `johnpaulabecia-production-nginx-1`
- Backend image observed: `ghcr.io/nohj-luap/johnpaulabecia-official-backend:07b1f1651c3a12303b64df180646f63e6e047bd9`
- Public endpoint: `https://api.sannycom.com/`
- Production Compose file: `/Applications/johnpaulabecia-apps/johnpaulabecia-official-devops/compose/docker-compose.production.yml`
- Backend environment file: `/Applications/johnpaulabecia-apps/johnpaulabecia-official-backend/.env.production`
- TLS certificate/key: `/etc/nginx/ssl/api.sannycom.com.pem` and `/etc/nginx/ssl/api.sannycom.com.key`, mounted into containerized Nginx

**Safety boundaries:** Do not remove Docker, Docker Compose, production images, `.ssh`, `.docker`, environment files, TLS files, or production Compose configuration. Existing local Docker images may be needed for deployment rollback. Do not use indiscriminate `docker system prune` or `docker image prune` commands.

## 2. Step-by-step execution record

### Step 1 — Establish baseline disk and service state

The initial `fastfetch` inspection showed approximately **4.31 GiB used / 6.61 GiB total (65%)**, around **453.60 MiB / 908.64 MiB RAM**, and no swap. The instance has a small root filesystem, so cleanup targeted obsolete host tools and caches rather than production Docker resources.

### Step 2 — Inspect APT's proposed orphan cleanup

Command recorded:

```bash
sudo apt autoremove --dry-run
```

Initial result: only `pollinate` was proposed. No packages were removed at this stage. The operator proceeded to investigate obsolete host Go and Nginx packages first.

### Step 3 — Verify old host-installed Go and Nginx are unused

Inspection commands and recorded observations:

```bash
command -v go
# /usr/bin/go

go version
# go1.26.0 linux/amd64

command -v nginx
# /usr/sbin/nginx

systemctl is-enabled nginx johnpaulabecia-backend
# disabled
# disabled

systemctl is-active nginx johnpaulabecia-backend
# inactive
# inactive
```

A `dpkg -l` package inspection identified six host packages: `golang-1.26-go`, `golang-1.26-src`, `golang-go`, `golang-src`, `nginx`, and `nginx-common`. The exact package-listing grep expression was not preserved in the available record.

**Decision:** Host Go and Nginx were obsolete after containerization. Both host services were disabled and inactive; production traffic was handled by Docker.

### Step 4 — Dry-run host package removal

```bash
sudo apt remove --dry-run \
  golang-go golang-src golang-1.26-go golang-1.26-src nginx nginx-common
```

APT proposed removal of exactly those six named packages, with no Docker package removal. The dry run also identified additional now-unneeded compiler/build dependencies that could later be removed by `autoremove`.

### Step 5 — Remove obsolete host Go and Nginx packages

```bash
sudo apt remove -y \
  golang-go golang-src golang-1.26-go golang-1.26-src nginx nginx-common
```

**Observed outcome:** Command succeeded; APT reported approximately **237 MB freed**. Docker packages were not removed.

### Step 6 — Verify production immediately after host package removal

Commands used:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

docker inspect --format 'Image: {{.Config.Image}}
Status: {{.State.Status}}
Health: {{.State.Health.Status}}' \
  johnpaulabecia-production-backend-1

curl -sS -o /dev/null \
  -w "HTTP Status: %{http_code}\n" \
  https://api.sannycom.com/

df -h /
```

**Observed:** Backend running and `healthy`; Nginx running with ports 80/443 exposed; backend image matched the SHA-tagged GHCR image; public HTTPS returned **HTTP 200**. Filesystem output at this checkpoint: `/dev/root 6.7G 4.1G 2.6G 62% /`.

### Step 7 — Remove orphaned build dependencies

First, dry-run:

```bash
sudo apt autoremove --dry-run
```

APT proposed **30** unused packages:

```text
cpp
cpp-15
cpp-15-x86-64-linux-gnu
cpp-x86-64-linux-gnu
g++
g++-15
g++-15-x86-64-linux-gnu
g++-x86-64-linux-gnu
gcc
gcc-15
gcc-15-base
gcc-15-x86-64-linux-gnu
gcc-x86-64-linux-gnu
libasan8
libcc1-0
libgcc-15-dev
libgomp1
libhwasan0
libisl23
libitm1
liblsan0
libmpc3
libpkgconf7
libquadmath0
libstdc++-15-dev
libtsan2
libubsan1
pkgconf
pkgconf-bin
pollinate
```

No Docker or SSH packages were included. Removal:

```bash
sudo apt autoremove -y
```

**Observed outcome:** All 30 packages removed; APT reported approximately **243 MB freed**. Together, the two APT operations reported roughly **480 MB** of package-space savings.

### Step 8 — Verify production and inspect Docker disk use

```bash
echo "=== Docker Containers ==="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

echo "=== Backend Health ==="
docker inspect \
  --format 'Status: {{.State.Status}}
Health: {{.State.Health.Status}}' \
  johnpaulabecia-production-backend-1

echo "=== Public API ==="
curl -sS -o /dev/null \
  -w "HTTP Status: %{http_code}\n" \
  https://api.sannycom.com/

echo "=== Disk Usage ==="
df -h /

echo "=== Docker Disk Usage ==="
docker system df
```

Actual output:

```text
=== Docker Containers ===
NAMES                                 STATUS                    PORTS
johnpaulabecia-production-backend-1   Up 32 minutes (healthy)   8080/tcp
johnpaulabecia-production-nginx-1     Up About an hour          0.0.0.0:80->80/tcp, [::]:80->80/tcp, 0.0.0.0:443->443/tcp, [::]:443->443/tcp
=== Backend Health ===
Status: running
Health: healthy
=== Public API ===
HTTP Status: 200
=== Disk Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       6.7G  3.9G  2.8G  58% /
=== Docker Disk Usage ===
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          4         3         142.7MB   16.47MB (11%)
Containers      2         2         73.73kB   0B (0%)
Local Volumes   0         0         0B        0B
Build Cache     0         0         0B        0B
```

**Decision:** Preserve the four local Docker images, including potentially useful rollback images. Only 16.47 MB of image storage was reclaimable; pruning posed unnecessary rollback risk.

### Step 9 — Identify remaining disk consumers

```bash
echo "=== APT Cache ==="
sudo du -sh /var/cache/apt/archives

echo "=== Systemd Journal ==="
sudo journalctl --disk-usage

echo "=== Temporary Files ==="
sudo du -sh /tmp /var/tmp

echo "=== Log Directory ==="
sudo du -sh /var/log

echo "=== Root Directory Usage ==="
sudo du -xhd1 / 2>/dev/null | sort -h
```

Actual output:

```text
=== APT Cache ===
820K /var/cache/apt/archives
=== Systemd Journal ===
Archived and active journals take up 24M in the file system.
=== Temporary Files ===
17M /tmp
56K /var/tmp
=== Log Directory ===
31M /var/log
=== Root Directory Usage ===
4.0K /media
4.0K /mnt
4.0K /srv
16K /lost+found
16K /opt
24K /snap
52K /root
6.5M /etc
19M /Applications
528M /home
823M /var
2.5G /usr
3.9G /
```

**Decision:** APT archives and journals were already small; `/home` (528 MB) merited investigation. No system logs or temporary files were deleted.

### Step 10 — Inspect home directory and Go caches

```bash
echo "=== Home Directory Usage ==="
du -h --max-depth=1 /home/ubuntu 2>/dev/null | sort -h

echo "=== Hidden Directories ==="
du -sh /home/ubuntu/.[!.]* 2>/dev/null | sort -h

echo "=== Go Installation and Cache ==="
du -sh \
  /home/ubuntu/go \
  /home/ubuntu/.cache/go-build \
  /usr/local/go \
  2>/dev/null || true
```

Actual output:

```text
=== Home Directory Usage ===
8.0K /home/ubuntu/.docker
12K /home/ubuntu/.local
28K /home/ubuntu/.ssh
72K /home/ubuntu/.config
147M /home/ubuntu/.cache
381M /home/ubuntu/go
528M /home/ubuntu
=== Hidden Directories ===
4.0K /home/ubuntu/.bash_logout
4.0K /home/ubuntu/.bashrc
4.0K /home/ubuntu/.profile
8.0K /home/ubuntu/.bash_history
8.0K /home/ubuntu/.docker
12K /home/ubuntu/.local
28K /home/ubuntu/.ssh
72K /home/ubuntu/.config
147M /home/ubuntu/.cache
=== Go Installation and Cache ===
381M /home/ubuntu/go
147M /home/ubuntu/.cache/go-build
```

**Decision:** Investigate the Go directory before deleting it. Preserve `.docker` and `.ssh`.

### Step 11 — Confirm contents of Go directories and active processes

```bash
echo "=== Go Directory Breakdown ==="
du -h --max-depth=2 /home/ubuntu/go 2>/dev/null | sort -h

echo "=== Installed Go Executables ==="
ls -lah /home/ubuntu/go/bin 2>/dev/null || true

echo "=== Go Build Cache ==="
du -sh /home/ubuntu/.cache/go-build

echo "=== Go-related Running Processes ==="
ps aux | grep -E '[g]o build|[g]o run|[g]opls|[a]ctionlint' || true
```

Actual output:

```text
=== Go Directory Breakdown ===
12K /home/ubuntu/go/pkg/sumdb
381M /home/ubuntu/go
381M /home/ubuntu/go/pkg
381M /home/ubuntu/go/pkg/mod
=== Installed Go Executables ===
=== Go Build Cache ===
147M /home/ubuntu/.cache/go-build
=== Go-related Running Processes ===
```

**Finding:** The Go directory was entirely module-cache content, with no `go/bin` executables listed. No matching Go build/tool processes were running.

### Step 12 — Remove obsolete Go caches; resolve permissions

Initial cleanup commands:

```bash
rm -rf /home/ubuntu/go/pkg/mod
rm -rf /home/ubuntu/.cache/go-build
```

**Observed failure:** The first command produced many `Permission denied` errors on files within module-cache directories, including files under `github.com/golang-jwt/jwt/v5@v5.3.1`. Go module-cache directories can be read-only, preventing recursive deletion by an ordinary user. The build cache was nevertheless removed successfully, as established by later verification.

**Resolution (actual successful command):**

```bash
sudo rm -rf -- /home/ubuntu/go/pkg/mod
```

This command completed without errors. It targeted only the previously inspected host module cache, not Docker storage.

**Post-cleanup verification (actual commands):**

```bash
echo "=== Remaining Go Cache ==="
du -sh /home/ubuntu/go /home/ubuntu/.cache 2>/dev/null || true

echo "=== Disk Usage ==="
df -h /

echo "=== Production Containers ==="
docker ps --format "table {{.Names}}\t{{.Status}}"

echo "=== Production API ==="
curl -sS -o /dev/null \
  -w "HTTP Status: %{http_code}\n" \
  https://api.sannycom.com/
```

Actual output:

```text
=== Remaining Go Cache ===
20K /home/ubuntu/go
16K /home/ubuntu/.cache
=== Disk Usage ===
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       6.7G  3.4G  3.3G  51% /
=== Production Containers ===
NAMES                                 STATUS
johnpaulabecia-production-backend-1   Up 36 minutes (healthy)
johnpaulabecia-production-nginx-1     Up About an hour
=== Production API ===
HTTP Status: 200
```

**Final explicit Go build-cache existence check:**

```bash
test -d /home/ubuntu/.cache/go-build \
  && echo "Go build cache exists" \
  || echo "Go build cache already removed"
```

Actual output:

```text
Go build cache already removed
```

## 3. Results and measurements

| Checkpoint | Used | Available | Root use |
|---|---:|---:|---:|
| Initial fastfetch baseline | ~4.31 GiB | ~2.3 GiB | 65% |
| After host Go/Nginx removal | 4.1G | 2.6G | 62% |
| After APT autoremove | 3.9G | 2.8G | 58% |
| After Go cache cleanup | **3.4G** | **3.3G** | **51%** |

APT reported **237 MB + 243 MB = 480 MB** freed. Go caches previously measured **381 MB + 147 MB = 528 MB**; their combined observed size was approximately 528 MB before deletion. Reported filesystem sizes are rounded, so these figures should not be treated as an exact byte-for-byte reconciliation. The overall disk-use reduction was approximately **14 percentage points**.

## 4. Issues encountered and resolution

| Issue | Evidence | Resolution |
|---|---|---|
| Host Go/Nginx might be needed | Host services were disabled/inactive; Docker production verified | Dry-run removal first, then remove only six named packages |
| APT could remove unrelated packages | Dry-run showed 30 specific orphan packages, no Docker/SSH packages | Run `sudo apt autoremove -y` only after review |
| Docker cleanup could break rollback | Four images, only 16.47 MB reclaimable | Preserve all local images; no Docker pruning |
| Go module cache refused deletion | Repeated `Permission denied` messages | Use `sudo rm -rf -- /home/ubuntu/go/pkg/mod` after path inspection |
| Uncertainty whether Go build cache was deleted | Directory check returned `Go build cache already removed` | No additional removal needed |
| Potential production disruption | Docker health and public HTTPS checks at several checkpoints | Backend remained `healthy`; endpoint returned HTTP 200 |

## 5. What was deliberately preserved

- Docker Engine, Compose, running containers, and GHCR images.
- Docker credentials in `/home/ubuntu/.docker`.
- SSH configuration/keys in `/home/ubuntu/.ssh`.
- Production Compose and backend environment files.
- Nginx TLS material under `/etc/nginx/ssl`.
- System logs, APT cache, and `/tmp` because their sizes did not justify additional risk.
- Production image history needed for the existing rollback strategy.

## 6. Final validation and operating guidance

At the final recorded checkpoint:

- Backend container: **running and healthy**.
- Nginx container: **running**.
- Public HTTPS endpoint: **HTTP 200**.
- Root filesystem: **6.7G total, 3.4G used, 3.3G available, 51% use**.
- Go module cache: **removed**.
- Go build cache: **removed and explicitly verified absent**.
- No container restart was needed during this cleanup.

For subsequent deployments, monitor `df -h /`, `docker system df`, container health, and the public endpoint. As new SHA-tagged images accumulate, assess rollback-image retention before deleting old images. Do not assume that an image marked *reclaimable* is safe to remove.

**Closure:** The obsolete V1 host deployment packages and Go caches were removed after V2 CI/CD containerization. Production remained available throughout the recorded verification steps, and the cleanup concluded without further deletion.
