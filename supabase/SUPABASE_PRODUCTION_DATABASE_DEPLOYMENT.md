# SUPABASE_PRODUCTION_DATABASE_DEPLOYMENT.md

## Supabase Production Database Deployment for the Go Backend

This document records the actual process completed for deploying the
PostgreSQL database of `johnpaulabecia-official-backend` to Supabase and
connecting the local Go backend to that production database.

It documents only the steps, configuration, commands, and verification
that were actually performed during the deployment process. It stops at
the completed local backend and database migration verification and does
not include the later AWS, Cloudflare, or Vercel deployment.

> **Security note:** The actual Supabase database password is
> intentionally not reproduced in this document.

------------------------------------------------------------------------

## 1. Supabase Project Setup

A Supabase project was created for the backend production database with
the following configuration:

  Setting             Value
  ------------------- --------------------------------------------
  Organization        `John Paul Abecia Projects`
  Project             `johnpaulabecia-official`
  Plan                Free
  Region selection    Asia-Pacific
  Project reference   `cqyyropjmlgzlegmkwsd`
  Project URL         `https://cqyyropjmlgzlegmkwsd.supabase.co`

The project later reported a **Healthy** status.

During setup, the following choices were used:

-   Data API: **OFF**
-   Automatically expose new tables: **OFF**
-   Automatic RLS: **OFF**

The Go backend was therefore intended to connect directly to PostgreSQL
rather than use Supabase's Data API.

------------------------------------------------------------------------

## 2. Initial Direct PostgreSQL Connection Attempt

The Supabase direct PostgreSQL connection was initially considered.

The direct endpoint was not usable from the local environment because
the connection resolved through IPv6 and the local connection attempt
failed.

We therefore did not use the direct endpoint as the working production
connection.

------------------------------------------------------------------------

## 3. Switch to the Supabase Session Pooler

We switched the database connection to the Supabase **Session Pooler**.

The actual working connection settings were:

  Setting    Value
  ---------- --------------------------------------------
  Host       `aws-0-ap-northeast-2.pooler.supabase.com`
  Port       `5432`
  Database   `postgres`
  User       `postgres.cqyyropjmlgzlegmkwsd`
  SSL mode   `require`

The database password is omitted from this document.

------------------------------------------------------------------------

## 4. Local Production Environment File

The production environment file was created in the local Go backend
repository as:

``` text
.env.production
```

Commands used while creating/editing the file included:

``` bash
touch .env.production
```

and:

``` bash
nano .env.production
```

The production configuration used was:

``` dotenv
APP_NAME=johnpaulabecia_portfolio
APP_ENV=production
APP_BASE_URL=http://localhost:8080
SERVER_PORT=8080

DB_HOST=aws-0-ap-northeast-2.pooler.supabase.com
DB_PORT=5432
DB_NAME=postgres
DB_USER=postgres.cqyyropjmlgzlegmkwsd
DB_PASSWORD='<REDACTED>'
DB_SSLMODE=require

DB_POOL_MAX_OPEN_CONNS=10
DB_POOL_MAX_IDLE_CONNS=2
```

The real database password contained shell-special characters. Its value
in `.env.production` was enclosed in single quotes so the file could be
loaded with `source`.

The repository's existing `.gitignore` rules ignored production
environment files through:

``` gitignore
.env
.env.*
!.env.example
```

Therefore, `.env.production` remained local rather than being committed
to the repository.

------------------------------------------------------------------------

## 5. Load the Production Environment Locally

The production environment variables were exported into the current
shell using:

``` bash
set -a; source .env.production; set +a
```

The database password variable was verified as being set without
printing its actual value.

Recorded verification:

``` text
DB_PASSWORD is set
```

------------------------------------------------------------------------

## 6. Verify the Go Backend Connection to Supabase

With `.env.production` loaded, the backend was started locally using:

``` bash
go run ./cmd/server
```

Recorded output:

``` text
[PRODUCTION] starting johnpaulabecia_portfolio
[PRODUCTION] environment: production
[PRODUCTION] database connection established
[PRODUCTION] server listening on :8080
```

This confirmed that the local Go backend successfully established its
PostgreSQL connection using the production Supabase configuration.

------------------------------------------------------------------------

## 7. Run the Existing Raw SQL Migrations Against Supabase

With the production environment loaded, the existing raw SQL migration
mechanism was executed using:

``` bash
go run ./cmd/migrate
```

Recorded output:

``` text
applied migration 000001_create_schema_migrations
applied migration 000003_create_users
applied migration 000004_create_communicate_messages
2026/10/05 23:40:11 database migrations completed successfully
```

The migrations applied to Supabase were:

1.  `000001_create_schema_migrations`
2.  `000003_create_users`
3.  `000004_create_communicate_messages`

The resulting tables were subsequently visible in the Supabase Table
Editor.

------------------------------------------------------------------------

## 8. Final Local Production Verification

The final backend startup verification was:

``` text
[PRODUCTION] starting johnpaulabecia_portfolio
[PRODUCTION] environment: production
[PRODUCTION] database connection established
[PRODUCTION] server listening on :8080
```

At this point, the verified database path was:

``` text
Local Go Backend
        |
        | PostgreSQL / TLS
        v
Supabase Session Pooler
aws-0-ap-northeast-2.pooler.supabase.com:5432
        |
        v
Supabase PostgreSQL
johnpaulabecia-official
```

The Supabase production database setup, local `.env.production`
configuration, backend database connection, and raw SQL migrations were
verified successfully.

------------------------------------------------------------------------

## 9. Commands Actually Used --- Consolidated Reference

``` bash
touch .env.production
nano .env.production
set -a; source .env.production; set +a
go run ./cmd/server
go run ./cmd/migrate
```

Successful backend connection output:

``` text
[PRODUCTION] starting johnpaulabecia_portfolio
[PRODUCTION] environment: production
[PRODUCTION] database connection established
[PRODUCTION] server listening on :8080
```

Successful migration output:

``` text
applied migration 000001_create_schema_migrations
applied migration 000003_create_users
applied migration 000004_create_communicate_messages
2026/10/05 23:40:11 database migrations completed successfully
```
