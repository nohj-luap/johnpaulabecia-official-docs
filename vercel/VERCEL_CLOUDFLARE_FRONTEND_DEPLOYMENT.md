# VERCEL_CLOUDFLARE_FRONTEND_DEPLOYMENT.md

## Vercel + Cloudflare Production Deployment for the Next.js Website

This document records the actual process completed for deploying
`johnpaulabecia-official-website` from GitHub to Vercel and configuring
the required Cloudflare DNS records for the production website domain.

It is intentionally scoped to:

-   the website repository and production branch state before deployment
-   GitHub integration with Vercel
-   Vercel Hobby/free deployment and built-in CI/CD
-   Next.js project import and build configuration
-   production and preview environment variables
-   the generated Vercel production domain
-   adding `sannycom.com` and `www.sannycom.com`
-   the exact DNS records Vercel requested during this deployment
-   the corresponding Cloudflare DNS changes
-   Vercel SSL certificate provisioning
-   production-domain verification
-   verification of the Next.js same-origin `/api` proxy to the already
    deployed AWS Go backend
-   apex-domain redirect verification

It intentionally does **not** repeat the AWS EC2 backend deployment,
Supabase database deployment, or the Cloudflare Origin Certificate
configuration for `api.sannycom.com`. Those are documented separately.

------------------------------------------------------------------------

## 1. Production Website Repository

The frontend repository used for the production website was:

``` text
johnpaulabecia-official-website
```

The local repository path was:

``` text
~/Applications/johnpaulabecia-app/johnpaulabecia-official-website
```

The website stack was:

``` text
Next.js 16.3.8
React 19.2.8
TypeScript
Tailwind CSS 4
App Router
npm
```

The website repository was hosted on GitHub and displayed in Vercel as:

``` text
nohj-luap/johnpaulabecia-official-website
```

------------------------------------------------------------------------

## 2. Verify the Production Branch Before Deployment

Before importing the website into Vercel, the local Git repository was
verified.

The resulting Git state was:

``` text
On branch main
Your branch is up to date with 'origin/main'.
nothing to commit, working tree clean
```

Therefore, the repository's `main` branch was the source used for the
initial Vercel production deployment.

------------------------------------------------------------------------

## 3. Why GitHub Was Connected to Vercel

The website repository was connected directly from GitHub to Vercel.

The deployment did **not** use:

-   a Dockerfile;
-   Docker Compose;
-   a manually copied production build;
-   a custom GitHub Actions deployment workflow.

Instead, the GitHub repository itself was imported into Vercel.

The deployment model used was:

``` text
Local development
       |
       | git push
       v
GitHub repository
nohj-luap/johnpaulabecia-official-website
       |
       | Vercel Git integration
       v
Vercel CI/CD
       |
       | build + deploy
       v
Production website
```

For this personal website deployment, the Vercel **Hobby/free plan** was
used.

Vercel's Git integration provided the automated deployment pipeline.
With the project connected to GitHub and `main` acting as the production
branch, changes pushed to the production branch can trigger a new Vercel
production deployment.

This is why no separate GitHub Actions workflow or Docker deployment
pipeline was required for the frontend.

------------------------------------------------------------------------

## 4. Import the GitHub Repository into Vercel

Inside Vercel, the GitHub repository was selected and imported.

The resulting project configuration was:

  Setting                          Value
  -------------------------------- ---------------------------------------------
  Vercel team                      `nohj-luap's projects`
  Plan                             Hobby
  Git provider                     GitHub
  Repository                       `nohj-luap/johnpaulabecia-official-website`
  Production branch                `main`
  Vercel project name              `johnpaulabecia-official-website`
  Root Directory                   `./`
  Framework / Application Preset   Next.js

The default Vercel build and output settings for the detected Next.js
application were retained.

No custom Docker build was configured.

------------------------------------------------------------------------

## 5. Production Frontend API Architecture

The frontend had already been designed to use a same-origin API proxy.

The local frontend environment used:

``` dotenv
NEXT_PUBLIC_API_URL=/api
API_BACKEND_URL=http://localhost:8080
```

The two variables had different purposes.

### Browser-visible API path

``` dotenv
NEXT_PUBLIC_API_URL=/api
```

The browser calls the website itself through `/api`.

For production, this means requests use the same website origin:

``` text
https://www.sannycom.com/api/...
```

### Server-side backend target

Locally:

``` dotenv
API_BACKEND_URL=http://localhost:8080
```

For Vercel production, this had to point to the already deployed public
Go backend:

``` dotenv
API_BACKEND_URL=https://api.sannycom.com
```

The browser therefore did not need to connect directly to the Go
backend.

The intended production path was:

``` text
Browser
   |
   v
Vercel / Next.js
https://www.sannycom.com
   |
   | /api/...
   v
Next.js same-origin API proxy
   |
   v
https://api.sannycom.com
   |
   v
Cloudflare
   |
   v
AWS EC2 / Nginx / Go
   |
   v
Supabase PostgreSQL
```

No Supabase database credentials were added to the Next.js deployment.

------------------------------------------------------------------------

## 6. Configure Vercel Environment Variables

During Vercel project setup, the following variables were configured:

``` text
NEXT_PUBLIC_API_URL=/api
API_BACKEND_URL=https://api.sannycom.com
```

They were added for:

``` text
Production
Preview
```

Therefore, the deployed Next.js application used:

``` text
/api
```

as the browser-facing API path and:

``` text
https://api.sannycom.com
```

as the server-side production proxy target.

------------------------------------------------------------------------

## 7. Initial Vercel Deployment

After the repository and environment variables were configured, the
project was deployed.

The deployment completed successfully and Vercel displayed its
deployment success / congratulations page with the rendered website.

The generated Vercel production domain was:

``` text
johnpaulabecia-official-website.vercel.app
```

Vercel showed this domain with:

``` text
Valid Configuration
Production
```

At this stage, the Next.js website was already deployed through Vercel
before the custom domain was completed.

------------------------------------------------------------------------

# Custom Domain and Cloudflare DNS

## 8. Add the Custom Domain to Vercel

The custom domain added to the Vercel project was:

``` text
sannycom.com
```

During the setup, the option to redirect the apex domain to `www` was
selected.

Vercel consequently configured:

``` text
sannycom.com
    |
    | 308 redirect
    v
www.sannycom.com
```

Both hostnames were connected to the production deployment.

The resulting Vercel domain entries were:

``` text
sannycom.com
www.sannycom.com
johnpaulabecia-official-website.vercel.app
```

------------------------------------------------------------------------

## 9. Exact DNS Records Requested by Vercel

For this actual deployment, Vercel displayed the following
project-specific DNS target:

``` text
4d7d0cdd5ff03cf7.vercel-dns-017.com
```

Vercel requested:

``` text
CNAME @   4d7d0cdd5ff03cf7.vercel-dns-017.com
CNAME www 4d7d0cdd5ff03cf7.vercel-dns-017.com
```

The Vercel interface instructed that the proxy be disabled for these
records.

These project-specific values were used rather than replacing them with
a generic Vercel DNS target.

------------------------------------------------------------------------

## 10. Existing Cloudflare Context

The `sannycom.com` DNS zone was managed through Cloudflare.

Before the Vercel frontend migration, the zone included older Cloudflare
Tunnel-related records.

The old root `sannycom.com` record was associated with the previous:

``` text
dyp-resto-prod
```

Cloudflare Tunnel deployment.

That old tunnel was deleted during the migration.

The user later confirmed that the older `ssh1` and staging tunnel setup
was no longer needed, so those old routes were not part of the new
Vercel production architecture.

The already working backend record remained separate:

``` text
api.sannycom.com
A
52.62.254.55
Proxied
```

That record continued to route the API to AWS EC2 and was not replaced
by the Vercel frontend records.

------------------------------------------------------------------------

## 11. Configure the Vercel Records in Cloudflare

The production website records added in Cloudflare were:

### Apex

``` text
Type: CNAME
Name: sannycom.com
Target: 4d7d0cdd5ff03cf7.vercel-dns-017.com
Proxy status: DNS only
```

### WWW

``` text
Type: CNAME
Name: www
Target: 4d7d0cdd5ff03cf7.vercel-dns-017.com
Proxy status: DNS only
```

The Cloudflare proxy was therefore **disabled** for the two Vercel
frontend records.

The resulting relevant DNS architecture was:

``` text
sannycom.com
    |
    | CNAME / DNS only
    v
Vercel

www.sannycom.com
    |
    | CNAME / DNS only
    v
Vercel


api.sannycom.com
    |
    | A / Cloudflare proxied
    v
52.62.254.55
AWS EC2
```

This kept frontend and backend routing separate.

------------------------------------------------------------------------

## 12. Vercel Domain Validation

After the Cloudflare DNS records were updated, Vercel was allowed to
validate the custom domains and provision its certificate.

During this process, the Vercel domain page showed:

``` text
sannycom.com
Valid Configuration
```

while:

``` text
www.sannycom.com
Generating SSL Certificate
```

was still in progress.

No additional DNS changes were made while the certificate was being
generated.

After certificate provisioning completed, all three relevant Vercel
domains showed:

``` text
Valid Configuration
```

including:

``` text
sannycom.com
www.sannycom.com
johnpaulabecia-official-website.vercel.app
```

------------------------------------------------------------------------

## 13. Verify the Production `www` Website

After Vercel reported valid configuration, the production website was
tested from the local Ubuntu terminal.

Command:

``` bash
curl -I https://www.sannycom.com/
```

Actual response:

``` text
HTTP/2 200
accept-ranges: bytes
access-control-allow-origin: *
age: 75
cache-control: public, max-age=0, must-revalidate
content-disposition: inline
content-type: text/html; charset=utf-8
date: Tue, 06 Oct 2026 04:18:05 GMT
etag: "1a10a35bb259c966d1f6cd1bbd1307e1"
server: Vercel
strict-transport-security: max-age=63072000
vary: rsc, next-router-state-tree, next-router-prefetch, next-router-segment-prefetch
x-matched-path: /
x-nextjs-prerender: 1
x-nextjs-stale-time: 300
x-vercel-cache: HIT
x-vercel-id: sin1::cxkxd-1791260285207-af04d5148dbb
content-length: 20660
```

The important result was:

``` text
HTTP/2 200
server: Vercel
```

This verified that:

``` text
https://www.sannycom.com/
```

was being served successfully by the Vercel production deployment.

------------------------------------------------------------------------

## 14. Verify the Production Next.js API Proxy

The next verification tested the complete frontend-to-backend path
rather than calling `api.sannycom.com` directly.

Command:

``` bash
curl -i https://www.sannycom.com/api/communicate/messages
```

Actual response:

``` text
HTTP/2 200
cache-control: public, max-age=0, must-revalidate
cf-cache-status: DYNAMIC
cf-ray: a461dcfd1fefab55-SIN
content-type: application/json
date: Tue, 06 Oct 2026 04:18:38 GMT
nel: {"report_to":"cf-nel","success_fraction":0.0,"max_age":604800}
server: Vercel
strict-transport-security: max-age=63072000
vary: Origin
x-vercel-cache: MISS
x-vercel-id: sin1::9ppxc-1791260317149-3d703c4872a9
content-length: 3

[]
```

The important result was:

``` text
HTTP/2 200
server: Vercel

[]
```

The presence of the successful JSON response through the website's
`/api` route verified the intended production path:

``` text
Client
   |
   v
https://www.sannycom.com
   |
   v
Vercel / Next.js
   |
   | /api/communicate/messages
   v
Next.js API proxy
   |
   v
https://api.sannycom.com
   |
   v
Cloudflare
   |
   v
AWS EC2 / Nginx
   |
   v
Go backend
   |
   v
Supabase PostgreSQL
```

------------------------------------------------------------------------

## 15. Verify the Apex Redirect

The apex domain was then tested.

Command:

``` bash
curl -I https://sannycom.com/
```

Actual response:

``` text
HTTP/2 308
cache-control: public, max-age=0, must-revalidate
content-type: text/plain
date: Tue, 06 Oct 2026 04:18:59 GMT
location: https://www.sannycom.com/
refresh: 0;url=https://www.sannycom.com/
server: Vercel
strict-transport-security: max-age=63072000
x-vercel-id: sin1::w9tlz-1791260338934-20ab9f3762e2
```

The important result was:

``` text
HTTP/2 308
location: https://www.sannycom.com/
server: Vercel
```

This verified the Vercel redirect configuration:

``` text
https://sannycom.com
        |
        | 308
        v
https://www.sannycom.com
```

------------------------------------------------------------------------

## 16. GitHub → Vercel CI/CD Result

The frontend deployment did not require a custom CI/CD server.

The website's GitHub repository was connected directly to Vercel:

``` text
GitHub
nohj-luap/johnpaulabecia-official-website
        |
        | production branch: main
        v
Vercel Git integration
        |
        | automatic build/deployment
        v
Vercel production
        |
        v
sannycom.com / www.sannycom.com
```

For the production workflow, updates to the website can therefore follow
the normal Git process:

``` bash
git add .
git commit -m "<commit message>"
git push origin main
```

Once the new commit reaches the connected `main` branch, Vercel's Git
integration is responsible for building and deploying the updated
production website.

No separate GitHub Actions deployment workflow was created for this
frontend deployment.

No Dockerfile was required for the Vercel-hosted Next.js frontend.

------------------------------------------------------------------------

## 17. Final Verified Production Architecture

At the end of this deployment phase, the verified full application path
was:

``` text
User
 |
 | HTTPS
 v
sannycom.com
 |
 | 308
 v
www.sannycom.com
 |
 v
Vercel CDN / Next.js
 |
 | same-origin /api
 v
Vercel Next.js API proxy
 |
 | HTTPS
 v
api.sannycom.com
 |
 v
Cloudflare
 |
 | HTTPS
 v
AWS EC2 / Nginx
 |
 v
Go backend
 |
 | PostgreSQL / TLS
 v
Supabase PostgreSQL
```

The frontend production host was served by Vercel.

The backend production host remained:

``` text
api.sannycom.com
```

and continued to use the separate Cloudflare → AWS EC2 architecture
documented in `AWS_EC2_BACKEND_DEPLOYMENT.md`.

------------------------------------------------------------------------

## 18. Commands Actually Used for Final Verification

### Verify the Vercel-hosted website

``` bash
curl -I https://www.sannycom.com/
```

Verified:

``` text
HTTP/2 200
server: Vercel
```

### Verify the frontend same-origin API proxy

``` bash
curl -i https://www.sannycom.com/api/communicate/messages
```

Verified:

``` text
HTTP/2 200
server: Vercel

[]
```

### Verify apex-domain redirect

``` bash
curl -I https://sannycom.com/
```

Verified:

``` text
HTTP/2 308
location: https://www.sannycom.com/
server: Vercel
```

------------------------------------------------------------------------

## 19. Deployment Completion State

The Vercel + Cloudflare frontend deployment was considered successfully
verified when all of the following had been demonstrated:

-   the website repository was clean and on `main`;
-   the GitHub website repository was imported into Vercel;
-   Vercel Hobby/free hosting was used;
-   the project was detected as Next.js;
-   root directory remained `./`;
-   the production branch was `main`;
-   no Dockerfile was required;
-   no custom GitHub Actions deployment workflow was required;
-   `NEXT_PUBLIC_API_URL=/api` was configured;
-   `API_BACKEND_URL=https://api.sannycom.com` was configured;
-   the environment variables were set for Production and Preview;
-   the generated `johnpaulabecia-official-website.vercel.app`
    deployment succeeded;
-   `sannycom.com` and `www.sannycom.com` were connected to production;
-   the exact Vercel-provided CNAME target was added in Cloudflare;
-   both frontend Vercel DNS records were configured as DNS only;
-   the existing `api.sannycom.com` AWS backend record remained
    separate;
-   Vercel completed SSL certificate generation;
-   all Vercel domains showed Valid Configuration;
-   `https://www.sannycom.com/` returned HTTP `200`;
-   `https://www.sannycom.com/api/communicate/messages` returned HTTP
    `200` and `[]`;
-   `https://sannycom.com/` returned HTTP `308` to
    `https://www.sannycom.com/`.

At that point, the website and its production API integration were live
and verified end-to-end.
