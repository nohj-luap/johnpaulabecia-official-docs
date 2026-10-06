# AWS_EC2_BACKEND_DEPLOYMENT.md

## AWS EC2 Production Deployment for the Go Backend

This document records the actual process completed for deploying
`johnpaulabecia-official-backend` to an AWS EC2 Ubuntu instance and
exposing the production API through Nginx and the Cloudflare
configuration required specifically for `api.sannycom.com`.

It is intentionally scoped to:

-   AWS account/free-plan and billing setup context used during
    deployment
-   EC2 instance creation
-   EC2 networking and Elastic IP
-   EC2 security group configuration
-   SSH access
-   Go backend production deployment
-   `.env.production` placement on EC2
-   systemd service setup
-   Nginx reverse proxy
-   Cloudflare DNS required for `api.sannycom.com`
-   Cloudflare Origin Certificate for the backend
-   Cloudflare `Full (strict)` TLS
-   production API verification

It intentionally excludes the later Vercel frontend deployment and the
Cloudflare configuration for `sannycom.com` and `www.sannycom.com`,
which belongs in a separate frontend/Vercel deployment document.

> **Security note:** Private keys, database passwords, payment-card
> details, and other credentials are intentionally not reproduced.

------------------------------------------------------------------------

## 1. AWS Account and Billing Context

An AWS account was created for the deployment.

The account was displayed as:

``` text
Caffeine Driven Development
```

During the AWS signup process, the required account and billing/payment
verification steps were completed. Payment-card details are not recorded
in this documentation.

The account was created under AWS's new credit-based Free plan context
rather than relying on the older EC2 12-month Free Tier model.

After signup, the AWS console showed:

``` text
Credits remaining: $100.00 USD
Days remaining: 183 days (Apr 06, 2027)
```

The deployment therefore proceeded with the initial **USD \$100 AWS
credit** and an approximately **six-month Free plan period**.

AWS currently documents that new customers receive USD \$100 in signup
credits and can earn additional credits, while the Free plan ends after
six months or when available credits are fully used, whichever occurs
first.

The deployment did not treat the EC2 server as permanently free.

------------------------------------------------------------------------

## 2. AWS Region

The EC2 backend was deployed in:

``` text
Sydney
ap-southeast-2
```

The created instance was placed in Availability Zone:

``` text
ap-southeast-2c
```

------------------------------------------------------------------------

## 3. EC2 Instance Creation

The EC2 instance was created with the following resulting configuration:

  Setting             Value
  ------------------- -----------------------------------
  Name                `johnpaulabecia-official-backend`
  Instance ID         `i-013b09e8c6539461c`
  Region              `ap-southeast-2`
  Availability Zone   `ap-southeast-2c`
  AMI / OS            Ubuntu Server 26.04 LTS amd64
  Instance type       `t3.micro`
  vCPU                2
  Memory              1 GiB
  Storage             approximately 8 GiB gp3
  Private IPv4        `172.31.20.54`

Termination protection was enabled for the instance.

The server later reported the AWS Ubuntu kernel:

``` text
7.0.0-1014-aws
```

------------------------------------------------------------------------

## 4. SSH Key Pair

The EC2 SSH private key used locally was:

``` text
~/.ssh/johnpaulabecia-official-backend.pem
```

The private key contents were never required to be copied into the
server or committed to the backend repository.

After the final Elastic IP was assigned, SSH access used:

``` bash
ssh -i ~/.ssh/johnpaulabecia-official-backend.pem ubuntu@52.62.254.55
```

------------------------------------------------------------------------

## 5. Elastic IP

The instance initially had an automatically assigned public IPv4
address:

``` text
13.211.169.77
```

A static Elastic IP was then allocated and associated with the EC2
instance.

The final production Elastic IP became:

``` text
52.62.254.55
```

From that point forward, the deployment used:

``` text
52.62.254.55
```

rather than the previous automatically assigned public address.

The purpose of the Elastic IP in this deployment was to give the backend
EC2 instance a stable public IPv4 address for the backend DNS record.

------------------------------------------------------------------------

## 6. EC2 Security Group

The instance used the security group:

``` text
launch-wizard-1
sg-009267d61aa4788f1
```

The resulting inbound rules used during the completed deployment were:

  Type    Protocol     Port Source
  ------- ---------- ------ --------------------
  SSH     TCP            22 `112.211.91.97/32`
  HTTP    TCP            80 `0.0.0.0/0`
  HTTPS   TCP           443 `0.0.0.0/0`

Port `8080` was **not** exposed publicly.

Outbound traffic remained allowed.

This kept the Go application behind Nginx:

``` text
Internet
   |
   v
EC2 :80 / :443
   |
   v
Nginx
   |
   v
127.0.0.1:8080
   |
   v
Go backend
```

------------------------------------------------------------------------

## 7. Backend Deployment Directory

The production backend was placed under:

``` text
/opt/johnpaulabecia/backend
```

The deployed executable was:

``` text
/opt/johnpaulabecia/backend/johnpaulabecia-backend
```

The production environment file was placed at:

``` text
/opt/johnpaulabecia/backend/.env.production
```

The environment file permissions were restricted to:

``` text
600
```

The `.env.production` file contained the production configuration
already verified during the Supabase deployment process. Its database
password is intentionally omitted here.

------------------------------------------------------------------------

## 8. systemd Service

The Go backend was configured as a systemd service at:

``` text
/etc/systemd/system/johnpaulabecia-backend.service
```

The service configuration used was:

``` ini
[Unit]
Description=John Paul Abecia Official Backend
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/opt/johnpaulabecia/backend
EnvironmentFile=/opt/johnpaulabecia/backend/.env.production
ExecStart=/opt/johnpaulabecia/backend/johnpaulabecia-backend
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

The service was enabled and started through systemd.

The resulting backend process listened internally on:

``` text
:8080
```

------------------------------------------------------------------------

## 9. Verify the Backend Directly on EC2

Before exposing the backend through Nginx, the application was verified
from inside the EC2 server.

The local request used:

``` bash
curl http://127.0.0.1:8080/
```

The response was:

``` text
John Paul Abecia API
```

This verified that:

1.  the production Go executable was running;
2.  systemd was managing the backend;
3.  the application was listening on port `8080`;
4.  the backend was reachable locally from the EC2 host.

------------------------------------------------------------------------

## 10. Install and Enable Nginx

Nginx was installed on the Ubuntu EC2 server and enabled as the public
reverse proxy.

The installed Nginx version was:

``` text
nginx 1.28.3
```

The backend Nginx site configuration was created at:

``` text
/etc/nginx/sites-available/johnpaulabecia-backend
```

and enabled through a symbolic link under:

``` text
/etc/nginx/sites-enabled/
```

The configuration was validated using:

``` bash
sudo nginx -t
```

After successful validation, Nginx was reloaded using:

``` bash
sudo systemctl reload nginx
```

------------------------------------------------------------------------

## 11. Backend Nginx Reverse Proxy

The backend hostname was:

``` text
api.sannycom.com
```

Nginx proxied requests to the Go backend at:

``` text
http://127.0.0.1:8080
```

After HTTPS was configured, the final Nginx site configuration was:

``` nginx
server {
    listen 80;
    listen [::]:80;

    server_name api.sannycom.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name api.sannycom.com;

    ssl_certificate /etc/nginx/ssl/api.sannycom.com.pem;
    ssl_certificate_key /etc/nginx/ssl/api.sannycom.com.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

The configuration was again validated with:

``` bash
sudo nginx -t
```

and reloaded with:

``` bash
sudo systemctl reload nginx
```

------------------------------------------------------------------------

# Cloudflare Configuration Required for the AWS Backend

The following Cloudflare configuration was part of the AWS backend deployment because it was required to expose the EC2 API securely through `api.sannycom.com`.

The later Vercel, apex-domain, and `www` configuration is deliberately excluded from this section.

---

## 12. Cloudflare Backend DNS Record

A DNS record was created in the Cloudflare DNS zone for `sannycom.com`.

The backend hostname was:

```text
api.sannycom.com
```

The DNS record used the following configuration:

| Setting | Value |
|---|---|
| Type | `A` |
| Name | `api` |
| IPv4 address | `52.62.254.55` |
| Proxy status | `Proxied` |

The IPv4 address was the Elastic IP assigned to the AWS EC2 backend instance.

The resulting path was:

```text
api.sannycom.com
        |
        v
Cloudflare
        |
        v
52.62.254.55
        |
        v
AWS EC2
```

The DNS record pointed to the EC2 **Elastic IP**, not the instance's previous automatically assigned public IPv4 address.

The Cloudflare proxy remained enabled for `api.sannycom.com`.

---

## 13. Create and Install the Cloudflare Origin Certificate

A Cloudflare Origin Certificate was created specifically for the AWS backend origin:

```text
api.sannycom.com
```

This certificate was used to secure the HTTPS connection between Cloudflare and the Nginx server running on AWS EC2.

### 13.1 Create the Origin Certificate in Cloudflare

From the Cloudflare dashboard for `sannycom.com`, the Origin Certificate was created from the SSL/TLS configuration.

The certificate settings used during the deployment were:

```text
Hostname: api.sannycom.com
Private key type: RSA 2048
Certificate validity: 15 years
```

After creating the certificate, Cloudflare displayed two PEM-formatted values.

The first was the **Origin Certificate**:

```text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

The second was the corresponding **Private Key**:

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

The actual certificate and private-key contents are intentionally not reproduced in this documentation.

The generated values were mapped to the EC2 files as follows:

```text
Cloudflare Origin Certificate
        |
        v
/etc/nginx/ssl/api.sannycom.com.pem


Cloudflare Private Key
        |
        v
/etc/nginx/ssl/api.sannycom.com.key
```

The private key must remain private and must not be committed to Git or included in project documentation.

### 13.2 Create the SSL Directory on EC2

On the EC2 server, the Nginx SSL directory was:

```text
/etc/nginx/ssl/
```

The resulting directory permissions were:

```text
700
```

The certificate and private key were stored as:

```text
/etc/nginx/ssl/api.sannycom.com.pem
/etc/nginx/ssl/api.sannycom.com.key
```

The resulting ownership and permissions were:

```text
api.sannycom.com.pem  root:root  644
api.sannycom.com.key  root:root  600
```

This allowed Nginx to read the certificate and private key while restricting access to the private key.

### 13.3 Configure Nginx to Use the Origin Certificate

The HTTPS Nginx server block referenced the installed files:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name api.sannycom.com;

    ssl_certificate /etc/nginx/ssl/api.sannycom.com.pem;
    ssl_certificate_key /etc/nginx/ssl/api.sannycom.com.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

The Nginx configuration was validated using:

```bash
sudo nginx -t
```

After successful validation, Nginx was reloaded:

```bash
sudo systemctl reload nginx
```

At this point, Nginx was configured to terminate HTTPS using the Cloudflare Origin Certificate and forward backend requests internally to:

```text
http://127.0.0.1:8080
```

---

## 14. Open HTTPS on the EC2 Security Group

After the Origin Certificate and HTTPS Nginx configuration were prepared, HTTPS was allowed through the EC2 security group.

The inbound rule was:

```text
Type: HTTPS
Protocol: TCP
Port: 443
Source: 0.0.0.0/0
```

The Go application's port `8080` remained private and was **not** exposed through the EC2 security group.

Therefore, external requests entered through Nginx rather than connecting directly to the Go process:

```text
Internet
   |
   | TCP 443
   v
AWS EC2
   |
   v
Nginx
   |
   | localhost
   v
127.0.0.1:8080
   |
   v
Go backend
```

---

## 15. Cloudflare SSL/TLS Encryption Mode

After HTTPS was configured on the EC2 origin using the Cloudflare Origin Certificate, the Cloudflare SSL/TLS encryption mode was changed to:

```text
Full (strict)
```

The Cloudflare dashboard subsequently showed:

```text
Current encryption mode: Full (strict)
```

With this configuration, the connection path became:

```text
Client
   |
   | HTTPS
   v
Cloudflare
   |
   | HTTPS
   | Origin Certificate validation
   v
AWS EC2
   |
   v
Nginx :443
```

`Full (strict)` ensured that Cloudflare connected to the EC2 origin using HTTPS and validated the certificate presented by Nginx.

---

## 16. Verify HTTPS Through Cloudflare and EC2

After the DNS record, Origin Certificate, Nginx HTTPS configuration, EC2 security-group rule, and Cloudflare `Full (strict)` mode were in place, the public production backend was tested.

The command used was:

```bash
curl -i https://api.sannycom.com/
```

The request returned:

```text
HTTP/2 200
```

with the API response:

```text
John Paul Abecia API
```

This confirmed the complete HTTPS request path:

```text
Client
   |
   | HTTPS
   v
api.sannycom.com
   |
   v
Cloudflare
Full (strict)
   |
   | HTTPS
   v
AWS EC2 :443
   |
   v
Nginx
   |
   | reverse proxy
   v
127.0.0.1:8080
   |
   v
Go backend
```

The successful `HTTP/2 200` response confirmed that:

1. `api.sannycom.com` resolved through Cloudflare;
2. Cloudflare successfully connected to the AWS EC2 origin over HTTPS;
3. the Cloudflare Origin Certificate was accepted under `Full (strict)` mode;
4. Nginx successfully terminated the origin HTTPS connection;
5. Nginx successfully proxied the request to `127.0.0.1:8080`;
6. the Go backend successfully returned `John Paul Abecia API`.

At this point, the Cloudflare-to-AWS HTTPS configuration for the production backend was verified successfully.

------------------------------------------------------------------------

## 17. Verify HTTP to HTTPS Redirect

The HTTP endpoint was tested using:

``` bash
curl -I http://api.sannycom.com/
```

The response returned an HTTP `301` redirect with:

``` text
Location: https://api.sannycom.com/
```

This confirmed that plaintext HTTP traffic was redirected to HTTPS by
Nginx.

------------------------------------------------------------------------

## 18. Verify Production CRUD Through the Public API

After the public HTTPS endpoint was operational, the Communicate module
was verified against production.

The backend routes were:

``` text
POST   /communicate/messages
GET    /communicate/messages
GET    /communicate/messages/{id}
PUT    /communicate/messages/{id}
DELETE /communicate/messages/{id}
```

The initial collection request:

``` text
GET https://api.sannycom.com/communicate/messages
```

returned:

``` text
HTTP 200
[]
```

A production message with the content:

``` text
Hello from production
```

was then created.

The POST request returned HTTP `201`, producing record:

``` text
id: 2
```

The record was retrieved through:

``` text
GET /communicate/messages/2
```

and returned HTTP `200`.

The same record was updated through:

``` text
PUT /communicate/messages/2
```

and returned HTTP `200`.

The record was deleted through:

``` text
DELETE /communicate/messages/2
```

and returned HTTP `204`.

Finally:

``` text
GET /communicate/messages/2
```

returned HTTP `404` with:

``` json
{"error":"message not found"}
```

This completed the production CRUD verification.

------------------------------------------------------------------------

## 19. Final Verified AWS Backend Architecture

At the end of this deployment phase, the verified backend production
path was:

``` text
Client
   |
   | HTTPS
   v
api.sannycom.com
   |
   v
Cloudflare
Full (strict)
   |
   | HTTPS
   v
AWS EC2
52.62.254.55
   |
   v
Nginx :443
   |
   v
127.0.0.1:8080
   |
   v
Go backend
   |
   | PostgreSQL / TLS
   v
Supabase Session Pooler
   |
   v
Supabase PostgreSQL
```

The frontend/Vercel deployment was not part of this phase.

------------------------------------------------------------------------

## 20. Commands Recorded During the AWS Backend Deployment

### Connect to EC2

``` bash
ssh -i ~/.ssh/johnpaulabecia-official-backend.pem ubuntu@52.62.254.55
```

### Verify the Go backend internally

``` bash
curl http://127.0.0.1:8080/
```

Expected response:

``` text
John Paul Abecia API
```

### Validate Nginx configuration

``` bash
sudo nginx -t
```

### Reload Nginx

``` bash
sudo systemctl reload nginx
```

### Verify the public HTTPS API

``` bash
curl -i https://api.sannycom.com/
```

Expected status:

``` text
HTTP/2 200
```

Expected body:

``` text
John Paul Abecia API
```

### Verify HTTP redirect

``` bash
curl -I http://api.sannycom.com/
```

Expected result:

``` text
HTTP 301
Location: https://api.sannycom.com/
```

------------------------------------------------------------------------

## 21. Deployment Completion State

The AWS backend deployment was considered successfully verified when all
of the following had been demonstrated:

-   AWS account signup and billing verification completed.
-   AWS console showed the initial `$100.00 USD` credit and
    `183 days (Apr 06, 2027)` remaining.
-   `t3.micro` Ubuntu EC2 instance running in Sydney.
-   Elastic IP `52.62.254.55` associated with the instance.
-   SSH restricted to the configured source IP.
-   Ports `80` and `443` available publicly.
-   Port `8080` not exposed publicly.
-   Go backend managed by systemd.
-   Production `.env.production` stored on EC2 with restricted
    permissions.
-   Backend reachable internally at `127.0.0.1:8080`.
-   Nginx reverse proxy operational.
-   `api.sannycom.com` proxied through Cloudflare to the EC2 Elastic IP.
-   Cloudflare Origin Certificate installed on Nginx.
-   Cloudflare SSL/TLS mode set to `Full (strict)`.
-   Public HTTPS root endpoint returned HTTP `200`.
-   HTTP redirected to HTTPS.
-   Production Communicate CRUD create/read/update/delete flow completed
    successfully.

This was the final verified state of the AWS EC2 backend deployment
before proceeding to the separate Vercel frontend deployment phase.
