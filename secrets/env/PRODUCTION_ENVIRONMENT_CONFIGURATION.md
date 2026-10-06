# PRODUCTION_ENVIRONMENT_CONFIGURATION.md

## Production Environment Configuration

This document defines the production environment configuration used by
the backend and frontend applications.

The backend and frontend maintain separate `.env.production`
configurations because they run in different environments and have
different responsibilities.

------------------------------------------------------------------------

## 1. Backend Production Environment

Repository:

``` text
johnpaulabecia-official-backend
```

Environment file:

``` text
.env.production
```

Production configuration:

``` dotenv
# Application
APP_NAME=johnpaulabecia_portfolio
APP_ENV=production
APP_BASE_URL=http://localhost:8080
SERVER_PORT=8080

# PostgreSQL - Supabase
DB_HOST=aws-0-ap-northeast-2.pooler.supabase.com
DB_PORT=5432
DB_NAME=postgres
DB_USER=postgres.cqyyropjmlgzlegmkwsd
DB_PASSWORD='<SUPABASE_DATABASE_PASSWORD>'
DB_SSLMODE=require

# Database Pool
DB_POOL_MAX_OPEN_CONNS=10
DB_POOL_MAX_IDLE_CONNS=2
```

The actual database password is intentionally not stored in this
documentation.

### Backend Environment Responsibilities

The application settings configure the Go production server:

``` text
APP_NAME
APP_ENV
APP_BASE_URL
SERVER_PORT
```

The PostgreSQL settings configure the backend connection to the Supabase
PostgreSQL Session Pooler:

``` text
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
DB_SSLMODE
```

The database pool settings control the Go application's SQL connection
pool:

``` text
DB_POOL_MAX_OPEN_CONNS=10
DB_POOL_MAX_IDLE_CONNS=2
```

The backend connects using the following architecture:

``` text
Go Backend
    |
    | PostgreSQL / TLS
    v
Supabase Session Pooler
    |
    v
Supabase PostgreSQL
```

------------------------------------------------------------------------

## 2. AWS EC2 Backend Deployment

On the production EC2 server, the backend environment file is stored at:

``` text
/opt/johnpaulabecia/backend/.env.production
```

The file is loaded by the backend systemd service using:

``` ini
EnvironmentFile=/opt/johnpaulabecia/backend/.env.production
```

The production environment therefore follows:

``` text
systemd
   |
   v
.env.production
   |
   v
Go Backend
   |
   v
Supabase PostgreSQL
```

The environment file should not be committed to Git.

The production server copy should use restricted permissions:

``` bash
sudo chmod 600 /opt/johnpaulabecia/backend/.env.production
```

------------------------------------------------------------------------

## 3. Frontend Production Environment

Repository:

``` text
johnpaulabecia-official-website
```

Environment file:

``` text
.env.production
```

Production configuration:

``` dotenv
NEXT_PUBLIC_API_URL=/api
API_BACKEND_URL=https://api.sannycom.com
```

### Frontend Environment Responsibilities

`NEXT_PUBLIC_API_URL` defines the URL used by browser-side frontend
requests:

``` text
NEXT_PUBLIC_API_URL=/api
```

The browser therefore sends API requests through the same-origin Next.js
API path rather than directly contacting the Go backend.

`API_BACKEND_URL` defines the production backend destination used by the
server-side Next.js API proxy:

``` text
API_BACKEND_URL=https://api.sannycom.com
```

The resulting request path is:

``` text
Browser
   |
   | /api/...
   v
Next.js / Vercel
   |
   | API_BACKEND_URL
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
Go Backend
```

------------------------------------------------------------------------

## 4. Vercel Production Environment Variables

For the deployed frontend, the production values are configured in
Vercel as environment variables rather than requiring the local
`.env.production` file to be uploaded.

The required values are:

``` text
NEXT_PUBLIC_API_URL=/api
API_BACKEND_URL=https://api.sannycom.com
```

These values were configured for the appropriate Vercel deployment
environments.

The flow is:

``` text
Vercel Environment Variables
          |
          v
Next.js Build / Runtime
          |
          v
Production Website
```

------------------------------------------------------------------------

## 5. Backend and Frontend Separation

The backend database configuration must remain backend-only.

The frontend does not require:

``` text
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
DB_SSLMODE
```

The correct separation is:

``` text
Frontend
--------
NEXT_PUBLIC_API_URL
API_BACKEND_URL

        |
        | HTTPS API
        v

Backend
-------
APP_NAME
APP_ENV
APP_BASE_URL
SERVER_PORT

DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
DB_SSLMODE

DB_POOL_MAX_OPEN_CONNS
DB_POOL_MAX_IDLE_CONNS

        |
        | PostgreSQL / TLS
        v

Supabase
```

The frontend communicates with the backend API. Only the backend
communicates directly with the PostgreSQL database.

------------------------------------------------------------------------

## 6. Production Architecture

The complete production environment-variable flow is:

``` text
Browser
   |
   | NEXT_PUBLIC_API_URL=/api
   v
Vercel / Next.js
   |
   | API_BACKEND_URL=https://api.sannycom.com
   v
api.sannycom.com
   |
   v
Cloudflare
   |
   v
AWS EC2
   |
   v
Nginx
   |
   v
Go Backend :8080
   |
   | DB_HOST
   | DB_PORT
   | DB_NAME
   | DB_USER
   | DB_PASSWORD
   | DB_SSLMODE
   v
Supabase PostgreSQL
```

------------------------------------------------------------------------

## 7. Environment File Policy

Local environment files should remain excluded from Git.

Typical `.gitignore` configuration:

``` gitignore
.env
.env.*
!.env.example
```

This allows repositories to contain an `.env.example` template while
preventing actual environment files such as:

``` text
.env.development
.env.production
.env.local
```

from being committed.

A repository-safe backend example can use:

``` dotenv
APP_NAME=johnpaulabecia_portfolio
APP_ENV=production
APP_BASE_URL=http://localhost:8080
SERVER_PORT=8080

DB_HOST=<SUPABASE_POOLER_HOST>
DB_PORT=5432
DB_NAME=postgres
DB_USER=<SUPABASE_DATABASE_USER>
DB_PASSWORD=<SUPABASE_DATABASE_PASSWORD>
DB_SSLMODE=require

DB_POOL_MAX_OPEN_CONNS=10
DB_POOL_MAX_IDLE_CONNS=2
```

A repository-safe frontend example can use:

``` dotenv
NEXT_PUBLIC_API_URL=/api
API_BACKEND_URL=https://api.example.com
```

------------------------------------------------------------------------

## 8. Deployment Summary

Backend production environment:

``` text
Location:
AWS EC2

Runtime file:
/opt/johnpaulabecia/backend/.env.production

Database:
Supabase PostgreSQL

Database connection:
Supabase Session Pooler
```

Frontend production environment:

``` text
Location:
Vercel

Environment variables:
NEXT_PUBLIC_API_URL=/api
API_BACKEND_URL=https://api.sannycom.com
```

The resulting separation is:

``` text
Vercel
Frontend Environment
       |
       v
Cloudflare / AWS
Backend Environment
       |
       v
Supabase
Database Environment
```

This keeps frontend routing configuration separate from backend
application and database configuration while preserving the production
deployment architecture.
