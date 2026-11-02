# Self-hosted backend for multiple sites

## Goal

One self-hosted Java backend on the home server that serves several unrelated Astro sites. No new EC2 instances. Performance is not a priority; cost and simplicity are.

| Site | What it is | Backend needs |
|---|---|---|
| lab131.com | Private place to share work things with specific people | Login, per-user access to shared items |
| strongereveryday.fit | Client's PT business site | Contact/intake forms; later a workout tracker and food log |
| cjscabinetry.com | Dad's cabinetry business site | Contact/quote forms; maybe booking |
| jorgensons.org | Family genealogy research site | Login, edit suggestions with a review queue |
| garage131.com | Hosted on AWS | Calendly-style booking |
| speedrival.com | Hosted on GCP | Calendly-style booking |

Booking is likely to be useful on several of these, so build it once as a shared module.

## Constraints and environment

- **Server:** Dell R730 at home. Lots of CPU, RAM and disk, and little else is running on it. Capacity is not a constraint.
- **Network:** Business-class line. The ISP knows it is a business connection, so hosting public services is allowed. The line is not always stable; see [sitetwo-site.md](sitetwo-site.md#known-facts) for how it compares with SiteTwo.
- **Hosting targets:** The API can run at Home (the R730) or at SiteTwo, and either can be the active site. See [sitetwo-failover.md](sitetwo-failover.md).
- **Ports:** Some ports already forward from the router to the server for other services. Opening one more is acceptable if that is the right design.
- **Main risk:** A DDoS against the home IP. If the line goes down, everything at home goes down with it, not just these sites.
- **lab131 unique URLs:** Today the "private" sharing is an unguessable URL that can only be changed by redeploying. That is a workaround for having no backend. A real login replaces it, and the unique-URL scheme goes away.

## Decision summary

| Decision | Choice |
|---|---|
| Static site hosting | Keep the Astro sites on static hosting (Cloudflare Pages or Netlify). Only the API runs at home. |
| Getting traffic home | Cloudflare in front of everything. Prefer Cloudflare Tunnel; port 443 locked to Cloudflare IPs is the fallback. |
| Routing to a site | `api.<domain>` per site, all pointing at one app. The app picks the tenant from the `Host` header. |
| App shape | One Java app (Spring Boot) in Docker, with modules switched on per tenant. |
| Data | Postgres, one schema per tenant, with Flyway migrations. |
| API versioning | `/v1/...` path prefix |
| Auth | Session cookies per tenant, passwordless email login (magic links), invite-only where needed |
| Backups | Nightly `pg_dump` per schema, copied offsite |

## 1. Getting traffic to the home server

The DDoS concern decides this one: **the home IP should never appear in DNS for these domains.** Everything goes through Cloudflare, which absorbs attacks on its edge. Either of the two setups below does that.

### Option A: Cloudflare Tunnel (recommended)

`cloudflared` runs on the R730 and keeps an outbound connection open to Cloudflare. DNS for `api.<domain>` points at the tunnel, not at your IP.

- No new inbound port. Nothing on the router changes.
- The origin cannot be reached directly at all, even by someone who knows the IP.
- Free TLS, and it survives an IP change.
- One tunnel can serve all the domains. Each hostname is an ingress rule.

```yaml
# /etc/cloudflared/config.yml
tunnel: home-api
credentials-file: /etc/cloudflared/home-api.json
ingress:
  - hostname: api.lab131.com
    service: http://localhost:8080
  - hostname: api.strongereveryday.fit
    service: http://localhost:8080
  - hostname: api.cjscabinetry.com
    service: http://localhost:8080
  - hostname: api.jorgensons.org
    service: http://localhost:8080
  - hostname: api.garage131.com
    service: http://localhost:8080
  - hostname: api.speedrival.com
    service: http://localhost:8080
  - service: http_status:404
```

### Option B: Open port 443, behind Cloudflare's proxy

The DNS records are proxied (orange cloud), so the public sees Cloudflare's IPs. The router forwards 443 to a reverse proxy (Caddy) on the R730.

This only protects you if the origin refuses everything that is not Cloudflare:
- Firewall 443 so it accepts only [Cloudflare's IP ranges](https://www.cloudflare.com/ips/).
- Turn on Authenticated Origin Pulls (mTLS), so only Cloudflare can complete a handshake.
- Use a Cloudflare Origin CA certificate on Caddy.

Without those steps, anyone who finds the IP can bypass Cloudflare.

### The existing open ports

The services that already have ports open may already expose the home IP, for example through an unproxied DNS record, a mail server, or a certificate in transparency logs. Either option above protects the new API, but an attacker who finds the IP elsewhere can still flood the line. Audit what is already exposed:
- Make sure no DNS record anywhere shows the home IP.
- Consider moving the other services behind the tunnel too.

**Recommendation: Option A.** It needs no new port and gives the strongest origin hiding, and it is less to maintain than firewall rules that follow Cloudflare's IP list.

## 2. One app, many tenants

Each site calls its own API subdomain (`api.cjscabinetry.com`, and so on). All of them reach the same Spring Boot app on port 8080.

Why a subdomain per site instead of one shared API host:
- Auth cookies are first-party for each site (set on `.jorgensons.org` and so on). There are no third-party cookie problems.
- CORS stays simple: each tenant allows exactly one origin.
- The client sites never show your other domains.

### Tenant resolution

A servlet filter maps `Host` to a tenant before anything else runs. An unknown host gets a 404.

```yaml
# tenants.yml
tenants:
  lab131:
    hosts: [api.lab131.com]
    origin: https://lab131.com
    schema: lab131
    modules: [auth, shares, forms]
  strongereveryday:
    hosts: [api.strongereveryday.fit]
    origin: https://strongereveryday.fit
    schema: strongereveryday
    modules: [forms, bookings, fitness]
    notify: owner@strongereveryday.fit
  cjscabinetry:
    hosts: [api.cjscabinetry.com]
    origin: https://cjscabinetry.com
    schema: cjscabinetry
    modules: [forms, bookings]
  jorgensons:
    hosts: [api.jorgensons.org]
    origin: https://jorgensons.org
    schema: jorgensons
    modules: [auth, suggestions]
  garage131:
    hosts: [api.garage131.com]
    origin: https://garage131.com
    schema: garage131
    modules: [bookings]
  speedrival:
    hosts: [api.speedrival.com]
    origin: https://speedrival.com
    schema: speedrival
    modules: [bookings]
```

### garage131 (AWS) and speedrival (GCP)

These two stay where they are. Only booking comes from the home API. A Cloudflare Tunnel hostname has to be in a zone managed by Cloudflare, so there are three ways to wire them in:

| Option | How | Notes |
|---|---|---|
| **A. Move only DNS to Cloudflare (recommended)** | Move the zones from Route 53 and Cloud DNS to Cloudflare (free). Recreate the existing records pointing at AWS and GCP, and add `api.<domain>` for the tunnel. | Hosting doesn't change. Every site is wired the same way, and every API gets DDoS protection. Before cutting over, check that any AWS or GCP certificate and domain verification records come along. |
| B. Server-to-server | If these sites already have their own Java backends, those backends call the home API with a per-tenant API key. The browser never talks to the home server. | No DNS change. The booking UI lives in their existing app. Good fit if the backends already exist. |
| C. A shared hostname | Use a hostname on a domain that is already on Cloudflare, such as `garage131.api.lab131.com`, with CORS allowing garage131.com. | Works because booking needs no login cookies. It shows lab131 in network requests. |

Add a tenant field `auth: apikey` for option B so those tenants accept a key header instead of CORS from browsers.

The tenant object is held in request scope. It decides:
- which DB schema the connection uses (`SET search_path`),
- which modules respond (a disabled module returns 404),
- the CORS origin, and where notification email goes.

### Why one app and not one per site

The R730 could easily run a separate JVM per site, so capacity is not the reason. The reason is that six apps mean six deploys, six sets of dependency upgrades, and six copies of auth and forms code. One app with schema-per-tenant isolation keeps the data separate and the code in one place.

If a tenant ever needs real separation (say the PT business grows and needs its own uptime or compliance story), run a second instance of the same image with a `tenants.yml` that lists only that tenant. No code changes are needed.

## 3. Data

Run Postgres in Docker on the R730, with **one schema per tenant**.
- Each tenant's data lives in a separate namespace, so a missing `WHERE tenant_id = ?` cannot leak one client's data to another.
- Each tenant can be backed up, restored or deleted on its own.
- Flyway runs the migrations for each schema at startup. Shared modules share migration scripts, and each schema only gets the scripts for its enabled modules.

### Backups

- A nightly `pg_dump` of each schema, encrypted and copied offsite (Backblaze B2 or Cloudflare R2, which costs pennies at this size).
- Once SiteTwo is running it becomes the second backup location. Keep the cloud copy as well. See [sitetwo-failover.md](sitetwo-failover.md#data).
- A restore test now and then.
- This is required once a client's data (strongereveryday) is on the box.

## 4. API design

Versioning uses a path prefix. Add `/v2` only for breaking changes, and run both versions side by side until the sites have moved over.

```
POST /v1/forms/{formId}                  public form submission (contact, quote, intake)
GET  /v1/bookings/availability           open slots for a booking type
POST /v1/bookings                        book a slot
POST /v1/auth/login                      send a magic link to an email
GET  /v1/auth/callback?token=...         exchange the link for a session cookie
POST /v1/auth/logout
GET  /v1/me
GET  /v1/shares                          lab131: items shared with the current user
POST /v1/suggestions                     jorgensons: propose an edit to a person record
GET  /v1/suggestions?status=open         jorgensons: review queue (admin)
```

## 5. Modules

### forms (all sites)
- Generic submissions: form id, JSON payload, timestamp, IP hash.
- An email notification goes to the tenant's `notify` address.
- Cloudflare Turnstile on every public form to stop spam.
- Per-form field validation lives in config, so a new form needs no code change.

### auth (lab131, jorgensons, later strongereveryday)
- Passwordless email login: the user enters an email, gets a link, and clicking it sets an HttpOnly session cookie. There are no passwords to store or reset, which suits family members and clients.
- Users are invite-only per tenant. The admin adds an email to the tenant's users table.
- Roles per tenant: `admin`, `member`.
- If you later want Google login or SSO across sites, Keycloak runs comfortably on the R730, with one realm per tenant. That is more to learn and run, so start without it.

### shares (lab131)
- This replaces the unique-URL scheme.
- Items (links, files, notes) have an access list of users. Sharing or revoking is a DB change with no redeploy.
- Files go on R730 disk and are served through the API after an access check.

### bookings (garage131, speedrival, maybe cjscabinetry and strongereveryday)
- Calendly-style: booking types (duration, buffer), weekly availability rules, blackout dates, and slots computed on request.
- Confirmation and cancellation emails with an `.ics` attachment.
- Optional later: read your Google Calendar's free/busy to block slots.
- garage131 and speedrival are booking-only tenants (see section 2 for how they connect from AWS and GCP).

### suggestions (jorgensons)
- Signed-in family members propose a change: person id, field, current value, proposed value, source or reasoning.
- The suggestion goes into a review queue with the status open, accepted or rejected.
- Accepted suggestions are **not** written straight into the site data. The genealogy site is built from hand-transcribed section files, and ambiguity is never silently resolved. An accepted suggestion becomes a to-do: edit the section file, rerun the build, redeploy. The API keeps the suggestion history; the site stays static.

### fitness (strongereveryday, later)
- Workout log and food log per client user, with the trainer as admin.
- Health data is sensitive. Agree with the client on retention and who can see what before building it.

## 6. Running it on the R730

```yaml
# docker-compose.yml (sketch)
services:
  api:
    image: home-api:latest
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/app
      TENANTS_FILE: /config/tenants.yml
    volumes:
      - ./config:/config:ro
      - ./files:/data/files
    restart: unless-stopped
  db:
    image: postgres:16
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    restart: unless-stopped
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel run home-api
    volumes:
      - ./cloudflared:/etc/cloudflared:ro
    restart: unless-stopped
```

- Postgres is not published to the host network. Only `cloudflared` talks to the outside.
- Outbound email goes through a transactional provider (Postmark, Resend or SES), not from the home IP, which would end up in spam folders.
- Monitoring: an external uptime check on `api.<domain>/v1/health` (UptimeRobot's free tier is enough).
- The same compose file runs at SiteTwo ([sitetwo-failover.md](sitetwo-failover.md)).

## 7. Failure modes

| Failure | Effect | Mitigation |
|---|---|---|
| Home power or internet outage | API down; static sites stay up | Forms show "try again or email us"; a UPS on the R730; fail over to SiteTwo ([sitetwo-failover.md](sitetwo-failover.md)) |
| Home line unstable (drops, slow periods) | API slow or unreachable at times; static sites unaffected | Switch the active API to SiteTwo while the home line is unreliable ([sitetwo-failover.md](sitetwo-failover.md#planned-switchover-home-is-up-but-unreliable)) |
| DDoS on a site | Absorbed at Cloudflare | Origin hidden by the tunnel; Cloudflare rate limiting on form and login endpoints |
| DDoS on the home IP found through another service | Whole line saturated | Audit the existing open ports; move those services behind the tunnel |
| Disk failure or a bad migration | Data loss | Offsite nightly dumps; RAID on the R730; test restores |
| Form spam | Junk in the DB and inboxes | Turnstile; rate limits |

## 8. Build order

1. Move DNS for all six domains to Cloudflare (if not already there). For garage131 and speedrival, only DNS moves; hosting stays on AWS and GCP. Set up the tunnel with a `/v1/health` endpoint.
2. The app skeleton: tenant filter, `tenants.yml`, schema-per-tenant Flyway setup, CORS.
3. The **forms** module. Wire up contact forms on cjscabinetry and strongereveryday.
4. The **auth** module, then **shares** for lab131. Retire the unique URLs.
5. **suggestions** for jorgensons.org.
6. The **bookings** module. Add garage131 and speedrival as tenants.
6a. SiteTwo failover, any time after step 2 ([sitetwo-failover.md](sitetwo-failover.md#4-order-of-work)).
7. The **fitness** module for strongereveryday, once scope is agreed with the client.
8. Offsite backups and uptime monitoring. Do this alongside step 3, before any real data arrives.
