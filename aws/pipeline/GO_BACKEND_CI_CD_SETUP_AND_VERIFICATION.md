# Go Backend CI/CD — Setup, Troubleshooting, and Verified Deployment

**Project:** `johnpaulabecia-official-backend`  
**Source:** 22-page Gemini AI conversation export, `ci-cd-pipeline.pdf` (October 9, 2026)  
**CI:** GitHub Actions + Go 1.27.1 + PostgreSQL 15  
**CD:** GitHub Actions → SSH → AWS EC2 → Go binary → systemd  
**Public API:** `https://api.sannycom.com/`

> **Scope and evidence:** This guide reconstructs the steps, decisions, problems, fixes, and final verification **actually recorded** in the provided conversation. It distinguishes early proposals from the deployed result. The final successful CD used a **native Go binary managed by systemd**, **not Docker Compose**. A Compose-based CD workflow was proposed earlier but was not the verified implementation in this record. Secrets and private-key contents are intentionally omitted.

## 1. Architecture and prerequisites

```text
Git push / pull request
       │
       ├── CI: GitHub-hosted Ubuntu runner
       │       ├── PostgreSQL 15 service, host port 5433
       │       ├── Go formatting / vet / unit tests
       │       ├── Test-only SQL migrations
       │       └── Authentication + authorization integration tests
       │
       └── CD: push to main
               └── GitHub Actions SSH → EC2 Ubuntu
                        ├── git pull origin main
                        ├── go build → native executable
                        └── systemctl restart backend
                                   │
                            Nginx on EC2 host
                                   │
                      Cloudflare Full (strict) HTTPS
                                   │
                         https://api.sannycom.com
                                   │
                         Supabase PostgreSQL
```

Recorded EC2 configuration: Ubuntu 26.04 LTS, `t3.micro` (2 vCPU, 1 GiB RAM), Sydney `ap-southeast-2`, Availability Zone `ap-southeast-2c`, Elastic IP `52.62.254.55`. Nginx is installed **on the VM**, not in Compose. The Go application listens on port `8080`; port `8080` is not publicly opened in the EC2 security group. The backend uses Supabase PostgreSQL in production.

**Repository branches:** `development`, `staging`, `preproduction`, `main`. CI was intentionally limited to `preproduction` and `main` to conserve private-repository Actions usage. CD runs on `main` pushes.

## 2. Phase I — Continuous Integration (CI)

### 2.1 Define the checks

The recorded local validation commands were:

```bash
gofmt -w .
go vet ./...
go test ./...
go run ./cmd/migrate
```

`gofmt -w .` **modifies files locally**. CI instead uses `gofmt -l .` to fail on unformatted source without rewriting the checkout.

Database-backed integration suites:

```bash
set -a
source .env.test
set +a

go test -count=1 -race -v -tags=integration \
  ./internal/modules/identity/authentication/integration

go test -count=1 -race -v -tags=integration \
  ./internal/modules/identity/authorization/integration
```

### 2.2 Provide an isolated PostgreSQL database

The existing local test Compose service used `postgres:15`, `5433:5432`, database `johnpaulabecia_official_test_db`, and a dedicated test user. In GitHub Actions the team selected a **job-level PostgreSQL service container** instead of running the local test Compose file.

The service is started **before checkout steps**, so it cannot read `.env.test` directly from the repository at service-creation time. Its `POSTGRES_*` values must be supplied through workflow configuration, GitHub variables/secrets, or another available Actions expression.

**Recorded choice:** nonproduction, throwaway test credentials were written into the workflow service definition and matched `.env.test`. This is acceptable only for genuinely disposable test credentials; never copy production credentials into the repository.

### 2.3 Resolve the ignored `.env.test` file

The `.env.test` file was initially ignored and therefore absent from GitHub. The conversation considered storing it as a GitHub Actions secret (`DOTENV_TEST`), but ultimately chose to commit the test-only configuration.

The recorded commands were:

```bash
# Adjust .gitignore so .env.test is not excluded, or explicitly force-add it.
git add -f .env.test
git commit -m "ci: track .env.test for automated CI integration tests"
git ls-files .env.test
```

**Observed result:** commit `05be0f7` created `.env.test` as a tracked file; `git ls-files .env.test` printed `.env.test`.

`.env.test` included application settings, PostgreSQL connection settings, connection pool limits, test JWT signing configuration, cookie settings, CORS origins, and an authorization admin-email setting. These were retained because the authentication/authorization tests initialize application configuration beyond the database connection.

**Security qualification:** The original PDF prints test credential values, a test JWT secret, and a personal email address. This documentation intentionally does **not** reproduce them. Verify that committed values are truly test-only; rotate any secret reused outside testing. A safer alternative is to keep `.env.test` ignored and provision it using GitHub Secrets or generate non-sensitive test defaults in CI.

### 2.4 Create `.github/workflows/ci.yml`

The following is a clean, syntactically organized reconstruction of the workflow recorded in the PDF. Replace placeholder **test-only** values consistently in both the service and `.env.test`.

```yaml
name: Continuous Integration

on:
  push:
    branches:
      - main
      - preproduction
  pull_request:
    branches:
      - main
      - preproduction

jobs:
  test:
    name: Run Format, Unit, and Integration Tests
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: johnpaulabecia_official_test_db
          POSTGRES_USER: johnpaulabeciauser
          POSTGRES_PASSWORD: REPLACE_WITH_TEST_ONLY_PASSWORD
        ports:
          - 5433:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v4

      - name: Setup Go Environment
        uses: actions/setup-go@v5
        with:
          go-version: "1.27.1"
          cache: true

      - name: Verify Code Formatting (gofmt)
        run: |
          if [ -n "$(gofmt -l .)" ]; then
            echo "The following files are not formatted:"
            gofmt -l .
            exit 1
          fi

      - name: Run Static Analysis (go vet)
        run: go vet ./...

      - name: Run Unit Tests
        run: go test -v ./...

      - name: Run Database Migrations
        run: |
          set -a; source .env.test; set +a
          go run ./cmd/migrate

      - name: Run Authentication Integration Tests
        run: |
          set -a; source .env.test; set +a
          go test -count=1 -race -v -tags=integration ./internal/modules/identity/authentication/integration

      - name: Run Authorization Integration Tests
        run: |
          set -a; source .env.test; set +a
          go test -count=1 -race -v -tags=integration ./internal/modules/identity/authorization/integration
```

**Reproduction note:** The PDF's actual test password was the same in the CI service and `.env.test`. The placeholder above deliberately avoids publishing it again. Before running this example, replace the placeholder with the **same disposable value** used in `.env.test`, or supply a GitHub secret via `${{ secrets.TEST_DB_PASSWORD }}`. The PDF used `actions/checkout@v4` and `actions/setup-go@v5`; these are the recorded versions, not an assertion that they are the latest.

### 2.5 Track, commit, push, and verify CI

The `.github/` folder was initially listed as **untracked** in `git status`. The conversation staged it along with `.gitignore`, integration-test READMEs, and workspace changes:

```bash
git add .github/ .gitignore \
  internal/modules/identity/*/integration/README.md \
  johnpaulabecia-official-backend.code-workspace

git commit -m "ci: add GitHub Actions workflow and update integration documentation"
git push origin main
```

For a workflow-only change, the narrower commands are:

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add GitHub Actions workflow for tests and migrations"
git push origin main
```

**Verification performed:** GitHub repository → **Actions** → **Continuous Integration**. The user reported a green check, including a successful **Run Authorization Integration Tests** step. The conversation interpreted this as confirmation that the PostgreSQL service started, formatting/vet/unit tests passed, test migrations applied, and both named integration suites passed.

**Important limitation:** A green CI check proves the configured checks passed for that run. It does not prove unconfigured tests passed or guarantee production correctness.

### 2.6 Production migrations: intentionally separate

The recorded decision was:

- **CI:** automatically run `go run ./cmd/migrate` against the temporary PostgreSQL test database.
- **Production:** continue manually reviewing and applying migrations to Supabase, using a controlled process, rather than automatically running them on every deployment.
- **Optional future approach:** a manually triggered `workflow_dispatch` migration workflow with explicit approval.

Production migrations were **not** established as part of the final automatic CD pipeline.

## 3. Phase II — Continuous Deployment (CD)

### 3.1 Choose the actual deployment method

Early responses proposed `docker-compose.yml` on EC2 and a Compose-based `.github/workflows/cd.yml`. **This was not the implemented endpoint of the conversation.** The user retained a host-level Nginx and a host-level **systemd Go service**, with the EC2 host compiling the Go binary after each Git pull.

**Final recorded deployment directory:**

```text
/Applications/johnpaulabecia-apps/johnpaulabecia-official-backend
```

This replaced the earlier suggestion `/opt/johnpaulabecia/backend`. The original binary-based deployment had used `/opt/...`; the CI/CD setup introduced the new repository checkout directory. Keep all systemd paths and workflow paths consistent.

### 3.2 Confirm Git on EC2

```bash
git --version
```

**Observed:** `git version 2.53.0`.

Running `git status` in `/opt/johnpaulabecia/backend` initially returned:

```text
fatal: not a git repository (or any of the parent directories): .git
```

This was expected because that directory originally contained a manually uploaded binary, not a Git checkout.

### 3.3 Clone the private backend repository into the chosen directory

The user preferred `git clone` rather than `git init` + `fetch` + `reset`:

```bash
sudo mkdir -p /Applications/johnpaulabecia-apps
sudo chown -R ubuntu:ubuntu /Applications/johnpaulabecia-apps
cd /Applications/johnpaulabecia-apps

git clone git@github.com:johnpaulabecia/johnpaulabecia-official-backend.git
cd johnpaulabecia-official-backend
git status
```

**Historical distinction:** The PDF initially suggested cloning via `https://github.com/...`; because the repository is private, that produced a GitHub username prompt. The guide then explored PAT and SSH deploy-key authentication. The SSH URL above is the reusable form **after** the EC2 deploy key has been registered and verified.

Configure a **read-only GitHub deploy key** for this repository (or a scoped GitHub App credential) before noninteractive `git pull`. Keep the private key on EC2 with restrictive permissions; register only the corresponding public key with GitHub.

### 3.4 Set up SSH authentication from EC2 to GitHub

The PDF records this host SSH configuration:

```bash
cat >> ~/.ssh/config <<'EOF'
Host github.com
  HostName github.com
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
```

This assumes `~/.ssh/id_ed25519` exists and its public key has been registered with GitHub. Editing the SSH config alone does not grant access. Verify using `ssh -T git@github.com` and `git remote -v`. For an HTTPS remote, after SSH authentication is configured, use:

```bash
git remote set-url origin git@github.com:johnpaulabecia/johnpaulabecia-official-backend.git
```

### 3.5 Install Go and compile on EC2

The first build failed with `go: command not found`; Go was installed on EC2, and the next build succeeded.

```bash
cd /Applications/johnpaulabecia-apps/johnpaulabecia-official-backend
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
  go build -o johnpaulabecia-backend ./cmd/server
```

The GitHub Actions SSH session is noninteractive and may not inherit the interactive shell's `PATH`. The workflow was amended to include:

```bash
export PATH=$PATH:/usr/local/go/bin:/usr/bin
```

### 3.6 Store `.env.production` only on EC2

Final repository checkout location:

```text
/Applications/johnpaulabecia-apps/johnpaulabecia-official-backend/.env.production
```

The source recorded `chmod 600` for this file. It was ignored by Git and retained on the VM across normal pulls. Do not commit production credentials. The earlier deployment used `/opt/johnpaulabecia/backend/.env.production`; ensure the live service uses the **new** path consistently.

### 3.7 Configure systemd

Unit path: `/etc/systemd/system/johnpaulabecia-backend.service`.

The following is a **normalized example** using the final path. The PDF includes a provisional service example with a mismatched `EnvironmentFile` path; verify the live unit with `systemctl cat`.

```ini
[Unit]
Description=John Paul Abecia Official Backend Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/Applications/johnpaulabecia-apps/johnpaulabecia-official-backend
EnvironmentFile=/Applications/johnpaulabecia-apps/johnpaulabecia-official-backend/.env.production
ExecStart=/Applications/johnpaulabecia-apps/johnpaulabecia-official-backend/johnpaulabecia-backend
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now johnpaulabecia-backend
sudo systemctl status johnpaulabecia-backend --no-pager -l
sudo systemctl cat johnpaulabecia-backend
```

**Observed failure:** systemd returned `status=203/EXEC` because the executable was missing at the configured path. After Go was installed and the binary built in the new checkout, the service started and connected to the database.

### 3.8 Allow narrowly scoped noninteractive service restart

A GitHub Actions SSH session cannot answer a sudo password prompt. The recorded sudoers rule was:

```bash
echo 'ubuntu ALL=(ALL) NOPASSWD: /bin/systemctl restart johnpaulabecia-backend' \
  | sudo tee /etc/sudoers.d/systemctl-backend
sudo chmod 440 /etc/sudoers.d/systemctl-backend
sudo visudo -cf /etc/sudoers.d/systemctl-backend
```

The permission/validation commands after `tee` are recommended verification steps. Confirm the `systemctl` executable path on the host if the rule does not match.

### 3.9 Configure GitHub Actions secrets

GitHub → repository → **Settings → Secrets and variables → Actions**.

| Secret         | Purpose                                                        |
| -------------- | -------------------------------------------------------------- |
| `EC2_HOST`     | EC2 Elastic IP (`52.62.254.55` in the source)                  |
| `EC2_USERNAME` | SSH account (`ubuntu`), if used by workflow                    |
| `EC2_SSH_KEY`  | Private key for GitHub Actions runner → EC2 SSH authentication |

**Separate credentials:** EC2 → private GitHub repository requires a **different repository deploy key or scoped credential**. The GitHub Actions-to-EC2 SSH key alone does not authorize `git pull` on EC2.

### 3.10 Create the CD workflow

The PDF alternates between `.github/workflows/cd.yml` and `deploy.yml`; both are valid GitHub Actions filenames. The final discussion references `cd.yml`. The successful CD approach compiled the **native Go binary** and restarted **systemd**, not Docker Compose.

```yml
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

concurrency:
  group: production-backend-deployment
  cancel-in-progress: false

jobs:
  deploy:
    if: >-
      ${{
        github.event.workflow_run.conclusion == 'success' &&
        github.event.workflow_run.event == 'push' &&
        github.event.workflow_run.head_branch == 'main'
      }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        env:
          DEPLOY_SHA: ${{ github.event.workflow_run.head_sha }}
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          envs: DEPLOY_SHA
          script: |
            set -e

            cd /Applications/johnpaulabecia-apps/johnpaulabecia-official-backend

            git fetch origin main
            git checkout --detach "$DEPLOY_SHA"

            export PATH="$PATH:/usr/local/go/bin:/usr/bin"

            CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
              go build -o johnpaulabecia-backend ./cmd/server

            sudo systemctl restart johnpaulabecia-backend
            sudo systemctl is-active --quiet johnpaulabecia-backend

            curl --fail --silent --show-error \
              --retry 5 \
              --retry-delay 2 \
              --retry-connrefused \
              http://127.0.0.1:8080/
```

**Source fidelity:** The historical workflow used `git pull origin main`, `go build`, and `systemctl restart`. The `set -e`, `--ff-only`, and retrying `curl` are **recommended safeguards added to this reconstructed example**, not claims about the exact historical file. The recorded action version was `appleboy/ssh-action@v1.0.3`.

**CI/CD ordering limitation:** Both workflows run on `main` pushes, but the PDF does **not** demonstrate a CI-success dependency for CD. A green CI check is therefore not necessarily a prerequisite to a CD run. Future improvement: gate CD on successful CI and deploy the validated commit/artifact.

### 3.11 Commit, push, and monitor CD

```bash
git add .github/workflows/cd.yml
git commit -m "cd: add automated deployment workflow for AWS EC2"
git push origin main
```

The PDF also records later workflow changes to add the explicit `PATH` and support passwordless service restart. Monitor GitHub **Actions** for the deployment run.

## 4. Problems and resolutions

| Problem                             | What happened                                          | Resolution                                                                |
| ----------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------- |
| `.github/` untracked                | New workflow directory not yet staged                  | `git add .github/`, commit, push                                          |
| `.env.test` ignored                 | Git did not include test environment                   | `git add -f .env.test`; `git ls-files .env.test` confirmed tracking       |
| `fatal: not a git repository`       | Original `/opt/...` binary directory had no `.git`     | Use new `/Applications/...` clone                                         |
| GitHub username prompt              | Private repo accessed using HTTPS                      | Configure SSH deploy key/SSH remote or a scoped PAT                       |
| `status=203/EXEC`                   | Missing executable at systemd `ExecStart`              | Install Go, build binary, align unit paths                                |
| `go: command not found`             | Go initially absent from EC2                           | Install Go; subsequent build succeeded                                    |
| Go missing in automated SSH `PATH`  | Noninteractive shell did not load interactive profile  | Export `/usr/local/go/bin:/usr/bin` in CD script                          |
| Git `Permission denied (publickey)` | Suggested possible private-repo authentication failure | Register deploy key, configure `~/.ssh/config`, verify SSH authentication |
| `sudo: a password is required`      | Suggested possible noninteractive restart failure      | Narrow `NOPASSWD` sudoers rule                                            |
| `dial tcp ***:22: i/o timeout`      | **Confirmed** GitHub runner could not reach EC2 SSH    | Change security group inbound SSH rule, rerun failed job                  |
| Green workflow but unclear restart  | Earlier PID had not changed yet                        | Inspect `journalctl`; later logs showed successful restart and new PID    |

The PDF confirms the `203/EXEC` problem, missing Go installation, SSH timeout, and final successful restart. Some Git/SSH/sudo error cases were **hypotheses offered during troubleshooting**, not separately proven failures.

### 4.1 Security warning: SSH was opened to the internet

The EC2 security group originally restricted SSH to a single `/32` client address. GitHub-hosted runners could not reach port 22, resulting in `i/o timeout`. The user changed the inbound SSH rule to **`0.0.0.0/0`** and reran the workflow. This allowed the recorded deployment to proceed, **but it is not a good permanent security posture**.

**Recommended remediation (not shown as completed):** restrict SSH using a trusted self-hosted runner, VPN/private network, AWS Systems Manager, or a managed GitHub-runner IP allowlist. Continue requiring key authentication and monitor access.

## 5. Final verification — October 9, 2026

### 5.1 GitHub Actions status

The user reported a **green check** for the CD workflow. The response concluded the GitHub Actions runner successfully connected to EC2, pulled source, compiled the binary, and invoked `systemctl restart`.

### 5.2 Verify live systemd logs

```bash
sudo journalctl -u johnpaulabecia-backend -f
sudo journalctl -u johnpaulabecia-backend -n 50
sudo journalctl -u johnpaulabecia-backend --since "10 minutes ago"
sudo journalctl -u johnpaulabecia-backend -p err
sudo systemctl status johnpaulabecia-backend --no-pager -l
```

The **actual final log** recorded:

```text
2026-10-09 12:01:43 UTC  Stopping johnpaulabecia-backend.service
2026-10-09 12:01:43 UTC  Deactivated successfully
2026-10-09 12:01:43 UTC  Started johnpaulabecia-backend.service
2026-10-09 12:01:43 UTC  [PRODUCTION] starting johnpaulabecia_portfolio
2026-10-09 12:01:43 UTC  [PRODUCTION] environment: production
2026-10-09 12:01:44 UTC  [PRODUCTION] database connection established
2026-10-09 12:01:44 UTC  [PRODUCTION] server listening on :8080
```

**Process IDs:** previous PID `20209` → new PID `20512`. This is direct evidence of a successful restart and backend startup after the automated deployment.

An earlier `GET /api/account/authentication/me` returned HTTP 401 (consistent with an unauthenticated request); the root route returned HTTP 200. These are not signs of a failed deployment.

### 5.3 Endpoint checks

```bash
# On EC2
curl -i http://127.0.0.1:8080/

# From any internet-connected client
curl -i https://api.sannycom.com/
curl -I http://api.sannycom.com/
```

**Expected based on the earlier AWS deployment verification:** root response `John Paul Abecia API` with HTTP 200; HTTP redirects to HTTPS. The first part of the PDF documents earlier successful public HTTPS and Communicate CRUD tests. The **final CD section itself** confirms the GitHub Actions success and systemd logs, but **does not show a new post-CD full CRUD test**.

### 5.4 Why `.github/` no longer appears in `git status`

Git tracks **files**, not folders. Once `.github/workflows/ci.yml` and `.github/workflows/cd.yml` are committed and unchanged, a clean `git status` will not list them.

```bash
git status
git ls-files .github/
git log --oneline -- .github/workflows/
```

A nonempty `git ls-files .github/` confirms that the workflow files are tracked. Editing them later will cause Git to show modifications again.

## 6. Verified completion state

| Area                                               | Result supported by PDF                                                       |
| -------------------------------------------------- | ----------------------------------------------------------------------------- |
| CI on `main` / `preproduction`                     | **Passed**; green Actions run                                                 |
| PostgreSQL 15 service, formatting, vet, unit tests | **Passed** in CI run                                                          |
| Test migrations and authn/authz integration tests  | **Passed** in CI run                                                          |
| CD on `main`                                       | **Passed**; green Actions run                                                 |
| Private repository checkout and Go build on EC2    | **Operational by final deployment**                                           |
| systemd restart                                    | **Confirmed** by PID change and journal logs                                  |
| Supabase production database connection            | **Confirmed** in startup log                                                  |
| Public HTTPS and CRUD                              | **Verified in earlier EC2 deployment**, not separately rerun in final CD logs |
| Docker Compose deployment                          | **Discussed, not the final deployed approach**                                |
| CD waits for successful CI                         | **Not established**                                                           |
| SSH restricted after workaround                    | **Not established**; last recorded rule was `0.0.0.0/0`                       |

### Recommended future improvements (not yet verified)

1. Restrict SSH ingress immediately; consider self-hosted runners or AWS Systems Manager.
2. Make CD conditional on successful CI and deploy an immutable, tested artifact.
3. Use atomic binary replacement and a rollback strategy instead of compiling over the live executable.
4. Add public HTTPS health checks and smoke tests after each deployment.
5. Keep production migrations controlled and backed up; test schema compatibility before release.
6. Monitor available disk space and logs on the small EC2 root volume.

---

**Documentation provenance:** Reconstructed from `ci-cd-pipeline.pdf`, pages 1–22. Earlier AWS/Nginx/Cloudflare deployment context and later GitHub Actions setup were both consulted. Historical claims, provisional suggestions, and new recommendations are labeled separately; sensitive values are omitted.
