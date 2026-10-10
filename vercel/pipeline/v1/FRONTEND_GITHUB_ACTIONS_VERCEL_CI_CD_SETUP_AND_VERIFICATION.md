# Frontend GitHub Actions CI/CD and Vercel Deployment

## 1. Purpose and final architecture

This document records the setup performed for `johnpaulabecia-official-website`: GitHub Actions validates the Next.js frontend before a separate GitHub Actions workflow builds and deploys the verified commit to the existing Vercel production project. Vercel's automatic Git-triggered deployments are disabled through `vercel.json`.

```text
Push to main ──> Continuous Integration (ci.yml)
                     ├─ npm ci
                     ├─ npm run lint
                     └─ npm test
                           │
                 failure ──┴── success
                    stop         │
                                 v
                       Continuous Deployment (cd.yml)
                         ├─ Checkout CI-verified commit
                         ├─ vercel pull --environment=production
                         ├─ vercel build --prod
                         └─ vercel deploy --prebuilt --prod
                                 │
                                 v
                         Vercel production
```

Pull requests targeting `main` run CI but do not deploy. Pushes to `development` do not trigger these workflows. CI failure causes the deployment job to be skipped. The CD workflow is triggered by the completion of the named CI workflow and then checks for a successful **push** event on `main`.

## 2. Verified project information

| Item | Value |
|---|---|
| Repository | `johnpaulabecia/johnpaulabecia-official-website` |
| Frontend | Next.js 16.3.8, React 19.2.8 |
| Node.js | `24.17.0` via `.nvmrc` |
| npm observed locally | `11.13.0` |
| Vercel CLI observed locally | `63.1.2` |
| Vercel account | `nohj-luap` |
| Vercel team | `nohj-luaps-projects` |
| Vercel plan | Hobby |
| Vercel project | `johnpaulabecia-official-website` |
| Vercel project ID | `prj_AnwWi7RRIfdWG35Ki05rwlNCOup2` |
| Vercel organization ID | `team_7xsVhuXfcxYHY07wPlQuzsJ1` |
| Initial CI/CD commit | `18cc2af` |

Project and organization IDs are identifiers, not authentication secrets. The Vercel access token **is secret** and must never be committed, printed in logs, or included in documentation.

## 3. Prerequisites

- Existing Next.js project with `package.json`, `package-lock.json`, and `.nvmrc`.
- GitHub repository with GitHub Actions enabled.
- Existing Vercel project connected to the same repository.
- Node Version Manager (`nvm`) available locally.
- Permission to create Vercel access tokens and GitHub Actions repository secrets.

From the repository root:

```bash
cd ~/Applications/johnpaulabecia-app/johnpaulabecia-official-website
cat .nvmrc
```

Expected:

```text
24.17.0
```

Select the correct Node.js version before local validation:

```bash
nvm install
nvm use
node -v
npm -v
npm ci
npm run lint
npm test
```

`npm ci` does **not** automatically select the version from `.nvmrc`. GitHub Actions uses `actions/setup-node` to do so. If imports fail locally, first verify Node.js, then inspect the specific module-resolution or test errors; not every import failure is caused by a Node.js mismatch.

## 4. Disable automatic Vercel Git deployments

Create `vercel.json` in the repository root:

```bash
touch vercel.json
nano vercel.json
```

Final contents:

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "git": {
    "deploymentEnabled": false
  }
}
```

Verify:

```bash
cat vercel.json
```

This prevents Vercel's Git integration from initiating builds for pushes. It does **not** prevent explicit deployments made with Vercel CLI. Keep this file in version control. **Do not activate this setting before preparing a replacement deployment path**, or production updates may stop.

## 5. Install and authenticate Vercel CLI

Use the Node.js version selected above:

```bash
nvm use 24.17.0
node -v
npm install -g vercel
vercel --version
```

Observed CLI output:

```text
Vercel CLI 63.1.2
63.1.2
```

Log in using the same account that owns the existing Vercel project:

```bash
vercel login
vercel whoami
```

The login command opens a browser/device authorization flow. Never publish a token or reusable credential from this process.

Observed verification:

```text
Logged in as nohj-luap
Active team: nohj-luaps-projects (nohj-luap's projects)
Active team plan: Hobby
```

## 6. Link the existing Vercel project

Run from the frontend repository:

```bash
vercel link
```

Choose the existing project:

```text
Team: nohj-luaps-projects
Project: johnpaulabecia-official-website (linked by git)
```

Observed successful output:

```text
✓ Linked nohj-luaps-projects/johnpaulabecia-official-website
✓ Updated .env.local file
```

With Vercel CLI 63.1.2, this setup produced `.vercel/repo.json` rather than `.vercel/project.json`. Verify the file that actually exists:

```bash
ls -la .vercel/
cat .vercel/repo.json
```

Observed content:

```json
{
  "remoteName": "origin",
  "projects": [
    {
      "id": "prj_AnwWi7RRIfdWG35Ki05rwlNCOup2",
      "name": "johnpaulabecia-official-website",
      "directory": ".",
      "orgId": "team_7xsVhuXfcxYHY07wPlQuzsJ1"
    }
  ]
}
```

Do not assume `.vercel/project.json` must exist for this CLI version. The successful `vercel link` result and matching identifiers in `.vercel/repo.json` were the observed verification.

**Environment-file distinction:** The Vercel CLI updated `.env.local`; the project's existing `.env.production` is a separate local file. The deployment workflow obtains the **Vercel project's production environment** with `vercel pull --environment=production`. It does not depend on committing `.env.production` to GitHub. Ensure all required production variables are configured in the Vercel project.

## 7. Create the Vercel access token

1. Open <https://vercel.com/account/tokens>.
2. Select **Create Token**.
3. Name it `github-actions-frontend-ci`.
4. Choose a scope with access to the existing website project.
5. Select an expiration period (the initial setup used **30 days**).
6. Create and copy the token once; do **not** paste it into Git, documentation, or chat.

Renew the token before expiration and update the corresponding GitHub secret. A 30-day token created on October 10, 2026 would need replacement by approximately November 9, 2026.

## 8. Configure GitHub Actions repository secrets

In the repository, navigate to **Settings → Secrets and variables → Actions → Repository secrets → New repository secret**.

Create:

| Secret name | Value |
|---|---|
| `VERCEL_TOKEN` | The private access token created in Step 7 |
| `VERCEL_PROJECT_ID` | `prj_AnwWi7RRIfdWG35Ki05rwlNCOup2` |
| `VERCEL_ORG_ID` | `team_7xsVhuXfcxYHY07wPlQuzsJ1` |

Verify all three secret **names** appear in the GitHub repository settings. GitHub will not reveal stored secret values after creation.

## 9. Final Continuous Integration workflow

Path: `.github/workflows/ci.yml`

```yaml
name: Continuous Integration

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  contents: read

jobs:
  ci:
    name: Lint and Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Run Jest tests
        run: npm test
```

The `node-version-file` option reads `24.17.0` from `.nvmrc`; no `nvm install` or `nvm use` step is required on the GitHub runner. `npm ci` installs exactly from the lockfile. Any nonzero lint/test exit status fails CI.

## 10. Final Continuous Deployment workflow

Path: `.github/workflows/cd.yml`

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

concurrency:
  group: vercel-production
  cancel-in-progress: false

jobs:
  deploy:
    name: Deploy to Vercel
    if: >-
      ${{
        github.event.workflow_run.conclusion == 'success' &&
        github.event.workflow_run.event == 'push' &&
        github.event.workflow_run.head_branch == 'main' &&
        github.event.workflow_run.head_repository.full_name == github.repository
      }}

    runs-on: ubuntu-latest

    env:
      VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
      VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
      VERCEL_TOKEN: ${{ secrets.VERCEL_TOKEN }}

    steps:
      - name: Checkout verified commit
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_sha }}
          persist-credentials: false

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - name: Install Vercel CLI
        run: npm install --global vercel

      - name: Pull Vercel production environment
        run: vercel pull --yes --environment=production --token="$VERCEL_TOKEN"

      - name: Build production application
        run: vercel build --prod --token="$VERCEL_TOKEN"

      - name: Deploy production application
        run: vercel deploy --prebuilt --prod --token="$VERCEL_TOKEN"
```

**Why the gate works:** `workflow_run` listens for completion of the workflow named `Continuous Integration`. The `if` expression allows the deploy job only if the CI conclusion was `success`, the original event was `push`, the branch was `main`, and the source repository matches. The checkout uses `head_sha` from that successful CI run, so the build uses the validated commit rather than the current tip of the branch.

**Important operational limitations:**

- The CD workflow must be present on the repository's **default branch** for `workflow_run` triggering.
- `concurrency` with `cancel-in-progress: false` serializes deployments but does **not** guarantee that an older successful CI run can never deploy after a newer commit. For strict latest-commit-only production releases, add a remote `main` SHA check immediately before deployment and consider a deployment queue.
- Because `workflow_run` executes a privileged downstream workflow, avoid checking out and executing untrusted pull-request code. This configuration explicitly excludes PR-triggered runs, but protect `main` and review workflow changes.
- `npm install --global vercel` follows the latest published CLI. For fully reproducible builds, pin a vetted CLI version and periodically update it.
- `vercel build --prod` occurs on GitHub Actions; `vercel deploy --prebuilt --prod` uploads and deploys its output. Production environment variables must be maintained in Vercel.

## 11. Validate local Git state and ignored files

Check workflow files:

```bash
ls -la .github/workflows/
```

Expected filenames:

```text
ci.yml
cd.yml
```

Check changed files before committing:

```bash
git status --short
```

Observed before the first commit:

```text
 M .github/workflows/ci.yml
?? .github/workflows/cd.yml
?? vercel.json
```

Verify secret-bearing local files are ignored:

```bash
git check-ignore .env.local .env.production .vercel/repo.json
```

Observed:

```text
.env.local
.env.production
.vercel/repo.json
```

Verify Node.js version configuration:

```bash
cat .nvmrc
```

Expected:

```text
24.17.0
```

## 12. Commit and activate the workflows

From the repository root:

```bash
git add .github/workflows/ci.yml \
        .github/workflows/cd.yml \
        vercel.json

git commit -m "ci(cd): separate GitHub Actions CI and Vercel deployment workflows"

git status --short
git branch --show-current
```

Observed commit:

```text
[main 18cc2af] ci(cd): separate GitHub Actions CI and Vercel deployment workflows
 3 files changed, 67 insertions(+), 1 deletion(-)
 create mode 100644 .github/workflows/cd.yml
 create mode 100644 vercel.json
```

The working tree was clean and the active branch was `main`. The setup was then activated by pushing:

```bash
git push origin main
```

**Production change caution:** Pushing to `main` activates the new CI/CD workflow and publishes `vercel.json`. For subsequent changes, prefer a pull request with required CI status checks before merging to `main`.

## 13. Verify the pipeline in GitHub and Vercel

Open <https://github.com/johnpaulabecia/johnpaulabecia-official-website/actions>.

1. Find **Continuous Integration** for the pushed `main` commit.
2. Inspect **Lint and Test** and confirm checkout, Node.js setup, `npm ci`, ESLint, and Jest succeeded.
3. Find the downstream **Continuous Deployment** run.
4. Confirm the deploy job was allowed only after the CI run concluded successfully.
5. Inspect logs for `vercel pull`, `vercel build --prod`, and `vercel deploy --prebuilt --prod`.
6. Open Vercel's deployment dashboard and verify a successful production deployment of the intended commit.
7. Confirm no separate automatic Git-triggered Vercel deployment was initiated for the same push.

Expected scenarios:

| Event | CI | CD deployment |
|---|---|---|
| Push to `development` | No | No |
| Pull request targeting `main` | Yes | Skipped |
| Push to `main`, lint fails | Fails | Skipped |
| Push to `main`, tests fail | Fails | Skipped |
| Push to `main`, all checks pass | Succeeds | Runs |

**Observed result:** The user confirmed the setup worked and reported a Vercel deployment stage of approximately **13 seconds**. That observation does not measure the full CI + CD duration.

## 14. Why Vercel showed a roughly 13-second deployment

Previously, Vercel's Git integration built and deployed the Next.js app. Under the new workflow, GitHub Actions performs the production build with `vercel build --prod`; Vercel receives prebuilt output through `vercel deploy --prebuilt --prod`. Vercel's reported deployment duration can therefore be substantially shorter because the production build already occurred elsewhere.

```text
Total elapsed time ≈ CI duration + CD setup/build duration + Vercel deployment duration
```

These stages can include GitHub runner queue/startup overhead. The 13-second figure applies to the reported Vercel deployment stage, not the complete end-to-end pipeline. CDN (content delivery network) is distinct from CD (continuous deployment); this change does not disable Vercel's CDN.

## 15. Troubleshooting

**CI import errors:** Verify `node -v` matches `.nvmrc`; run `nvm use`, `npm ci`, and the failing command locally. Inspect module aliases, missing dependencies, TypeScript paths, and Jest configuration if errors persist.

**CI passes but CD does not run:** Confirm the upstream workflow name exactly matches `Continuous Integration`; verify the CD workflow exists on the default branch; inspect the original event type, branch, repository, and CI conclusion. PR success intentionally does not deploy.

**CD fails authentication:** Confirm `VERCEL_TOKEN` exists, is not expired, and has access to the project; check `VERCEL_PROJECT_ID` and `VERCEL_ORG_ID` against `.vercel/repo.json`.

**Vercel project link confusion:** Newer Vercel CLI versions may use `.vercel/repo.json` rather than `.vercel/project.json`. Confirm the `vercel link` success message and the identifiers actually present.

**Production environment mismatch:** Verify production variables in the Vercel dashboard. The `.env.production` file is intentionally Git-ignored and is not automatically available to GitHub Actions.

**Unexpected automatic Vercel build:** Confirm `vercel.json` was committed on the deployed branch and `git.deploymentEnabled` is `false`. Review Vercel's Git integration settings and deployment source.

**Older build overtakes newer build:** The recorded workflow does not implement a latest-commit check. Add one before deployment if strict ordering is required.

**Token expires:** Generate a replacement Vercel token and update `VERCEL_TOKEN` in GitHub Actions secrets; do not commit it.

## 16. Optional main-branch protection

For changes after initial activation, configure a GitHub branch ruleset or branch protection rule targeting `main` and require the `Lint and Test` status check before merging. This blocks merging failing PRs, subject to GitHub's repository-plan and private-repository enforcement rules. The push-triggered CI/CD remains a separate safety check after the merge. Avoid unrestricted direct pushes to `main` if pre-merge enforcement is required.

## 17. Final file layout

```text
johnpaulabecia-official-website/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── cd.yml
├── .nvmrc                  # 24.17.0
├── vercel.json             # disable automatic Git deployments
├── package.json
└── package-lock.json
```

### Reference documentation

- GitHub Actions `workflow_run`: <https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#workflow_run>
- GitHub Actions workflow syntax: <https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax>
- Vercel Git configuration: <https://vercel.com/docs/project-configuration/git-configuration>
- Vercel with GitHub Actions: <https://vercel.com/kb/guide/how-can-i-use-github-actions-with-vercel>
- Vercel CLI: <https://vercel.com/docs/cli>

---

**Implementation record:** The commands, identifiers, filenames, initial commit, and outcomes above are taken from the completed setup conversation. The YAML and JSON blocks reproduce the final configurations provided during that setup. Operational cautions and verification checks are included to support future maintenance.
