# CLOUDFLARE_DOMAIN_REGISTRAR_SETUP.md

## Cloudflare Billing and Domain Registrar Setup

This document provides a simple reusable process for setting up
Cloudflare billing, purchasing a domain through Cloudflare Registrar,
and organizing domain names for production and development environments.

This guide is intentionally limited to domain registration and
environment naming. Application deployment, AWS, Vercel, SSL origin
certificates, and application-specific DNS records should be documented
separately.

------------------------------------------------------------------------

## 1. Create or Sign In to a Cloudflare Account

Sign in to the Cloudflare dashboard using the account that will own and
manage the domains.

Before registering a domain, make sure the account email address is
verified.

Domains purchased through Cloudflare Registrar use Cloudflare
nameservers.

------------------------------------------------------------------------

## 2. Configure Billing and a Payment Method

Open the Cloudflare account billing settings and create or complete the
billing profile.

Add a valid payment method that will be used for domain registration and
future renewals.

Cloudflare currently supports payment methods including major cards and
other supported payment services depending on the account and region.

Do not store payment-card numbers, security codes, or other payment
credentials in project documentation or source-control repositories.

The resulting account flow is:

``` text
Cloudflare Account
        |
        v
Billing Profile
        |
        v
Default Payment Method
        |
        v
Cloudflare Registrar
```

A valid billing profile and payment method must be available before
completing a domain purchase.

------------------------------------------------------------------------

## 3. Open Cloudflare Registrar

From the Cloudflare dashboard, open the domain registration area and
select the option to register a new domain.

Search for the desired domain name.

Example:

``` text
example.com
```

Cloudflare will display whether the domain is available for registration
and show the applicable registration price.

If the exact domain is unavailable, choose another available domain or
extension.

------------------------------------------------------------------------

## 4. Review the Domain Before Purchasing

Before completing the registration, verify the domain carefully.

For example:

``` text
example.com
```

Check:

-   spelling;
-   domain extension such as `.com`;
-   registration price;
-   registration period;
-   renewal information;
-   registrant/contact information.

Domain registrations should be reviewed carefully before purchase
because successfully completed domain registrations are generally
non-refundable.

------------------------------------------------------------------------

## 5. Select the Registration Period

Select the desired registration term.

For many domains, a one-year registration can be selected, although
available registration periods depend on the domain extension.

The checkout page displays the resulting expiration date and price.

Cloudflare Registrar registrations made through the dashboard have
auto-renew enabled by default. Auto-renew can be changed later from the
domain management settings.

------------------------------------------------------------------------

## 6. Enter Registrant Information

Provide the required domain contact information.

This generally includes:

``` text
First name
Last name
Email
Phone number
Address
City
State / Province
Country
Postal code
```

Use accurate information because the registration information is
associated with ownership and management of the domain.

Cloudflare redacts registrant information from public WHOIS output when
permitted by the registry.

------------------------------------------------------------------------

## 7. Complete the Domain Purchase

Select the configured payment method and review the registration details
one final time.

Confirm:

``` text
Domain name
Registration term
Price
Registrant information
Payment method
Auto-renew setting
```

Accept the required domain-registration terms and complete the purchase.

After a successful registration, the domain becomes manageable from
Cloudflare's domain management interface.

A confirmation email is also sent for the registration.

Complete any required registrant-email verification when requested.

------------------------------------------------------------------------

# Production and Development Domain Organization

## 8. Recommended Simple Structure

A separate purchased domain is not normally required for every
environment.

For a simple application, one registered domain can be organized using
subdomains.

Example:

``` text
example.com
```

Production:

``` text
example.com
www.example.com
api.example.com
```

Development or staging:

``` text
dev.example.com
staging.example.com
api-dev.example.com
```

Example architecture:

``` text
example.com
|
+-- Production
|   |
|   +-- example.com
|   +-- www.example.com
|   +-- api.example.com
|
+-- Non-production
    |
    +-- dev.example.com
    +-- staging.example.com
    +-- api-dev.example.com
```

This allows one registered domain to support multiple application
environments without purchasing another domain.

------------------------------------------------------------------------

## 9. Production Domain

Use the primary domain for the public production application.

Example:

``` text
example.com
```

Common production hostnames include:

``` text
example.com
www.example.com
api.example.com
```

A possible structure is:

``` text
example.com
        |
        +-- www.example.com  -> production website
        |
        +-- api.example.com  -> production backend
```

The exact DNS targets depend on the hosting providers and should be
configured as part of the application's deployment documentation.

------------------------------------------------------------------------

## 10. Development and Staging Domains

For internal development, testing, staging, or pre-production
environments, create subdomains under the same registered domain.

Examples:

``` text
dev.example.com
staging.example.com
preproduction.example.com
```

Backend environments can similarly use:

``` text
api-dev.example.com
api-staging.example.com
```

This avoids paying for a completely separate registered domain simply to
create another environment.

------------------------------------------------------------------------

## 11. When to Purchase a Separate Development Domain

A second registered domain may be purchased if complete domain-level
isolation is intentionally required.

Example:

``` text
Production:
example.com

Development:
example-dev.com
```

This can be useful when the development environment must be completely
independent from the production DNS zone or when testing domain-level
behavior that cannot conveniently be isolated through subdomains.

However, purchasing a second domain creates another registration that
must be paid for, renewed, and managed.

For most small projects, the simpler structure is:

``` text
Production:
example.com

Development:
dev.example.com

Staging:
staging.example.com
```

------------------------------------------------------------------------

## 12. Example Based on a Production Website Architecture

A practical organization can look like:

``` text
Production
----------
example.com
www.example.com
api.example.com

Development
-----------
dev.example.com
api-dev.example.com

Staging
-------
staging.example.com
api-staging.example.com
```

Only:

``` text
example.com
```

needs to be registered as the domain.

The other names are subdomains created through DNS configuration.

------------------------------------------------------------------------

## 13. Domain Management After Purchase

After registration, use Cloudflare's domain management page to review:

``` text
Registration status
Expiration date
Auto-renew status
Registrant information
Registration fees
```

Cloudflare Registrar domain registrations are managed separately from
ordinary Cloudflare subscriptions.

Review the domain's renewal configuration before its expiration date.

------------------------------------------------------------------------

## 14. Recommended Documentation Boundary

Keep domain ownership and Registrar documentation separate from
application deployment documentation.

A clean documentation structure is:

``` text
CLOUDFLARE_DOMAIN_REGISTRAR_SETUP.md
    |
    +-- Cloudflare account
    +-- billing profile
    +-- payment method
    +-- domain search
    +-- registration
    +-- renewal
    +-- production/development naming

AWS_EC2_BACKEND_DEPLOYMENT.md
    |
    +-- EC2
    +-- backend DNS
    +-- Nginx
    +-- Cloudflare Origin TLS

VERCEL_CLOUDFLARE_FRONTEND_DEPLOYMENT.md
    |
    +-- GitHub
    +-- Vercel
    +-- frontend DNS
    +-- custom domain
    +-- frontend verification
```

This prevents domain ownership and billing procedures from becoming
mixed with provider-specific application deployment steps.

------------------------------------------------------------------------

## 15. Final Checklist

Before considering the Registrar setup complete, verify:

``` text
[ ] Cloudflare account email is verified
[ ] Billing profile is configured
[ ] Valid payment method is configured
[ ] Desired domain is available
[ ] Domain spelling and extension are correct
[ ] Registration price and term are reviewed
[ ] Registrant information is accurate
[ ] Domain purchase is completed
[ ] Required registrant email verification is completed
[ ] Auto-renew preference is reviewed
[ ] Production hostname structure is defined
[ ] Development/staging subdomains are defined if needed
```

A simple final domain strategy is:

``` text
Registered domain:
example.com

Production:
example.com
www.example.com
api.example.com

Development:
dev.example.com
api-dev.example.com

Staging:
staging.example.com
api-staging.example.com
```

This keeps domain registration simple while still allowing production,
development, and staging environments to remain logically separated.
