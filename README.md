# Prismatite

Planning docs for a small self-hosted infrastructure: one Java backend running on a home server that serves several unrelated websites, a standby site in Minnesota, and a Tablo (over-the-air DVR) that family can watch remotely. There is no code yet. These files are the design.

## The idea

Six websites need a backend for things a static site can't do (forms, logins, booking, and later a workout tracker). Instead of paying for new cloud servers or running one app per site, a single Spring Boot app runs on a Dell R730 at home. It serves every site, picking the right "tenant" from the request's `Host` header. The static Astro sites stay on static hosting, and only the API runs at home.

| Site | Backend needs |
|---|---|
| lab131.com | Login, per-user access to shared items |
| strongereveryday.fit | Contact/intake forms; later a workout tracker and food log |
| cjscabinetry.com | Contact/quote forms; maybe booking |
| jorgensons.org | Login, edit suggestions with a review queue |
| garage131.com (on AWS) | Calendly-style booking |
| speedrival.com (on GCP) | Calendly-style booking |

Cost and simplicity matter more than performance, and none of this is critical, so downtime is acceptable.

## Main design choices

- **Cloudflare Tunnel** in front of everything, so the home IP never appears in DNS. This is the defense against a DDoS on the home line.
- **One app, many tenants**, with modules (forms, auth, shares, bookings, suggestions, fitness) switched on per tenant.
- **Postgres with one schema per tenant**, with nightly `pg_dump` backups copied offsite.
- **Passwordless magic-link login**, invite-only.
- **SiteTwo**, a second location in Minnesota with a faster, steadier fiber line, acts as a cold standby. Failover is manual: restore the latest backup, start the API and repoint DNS. Automatic failover is deliberately avoided because of split-brain risk.

## The documents

| File | What it covers |
|---|---|
| [multiproject.md](multiproject.md) | The application: tenants, modules, API, data, traffic routing, failure modes and build order |
| [sitetwo-failover.md](sitetwo-failover.md) | Running the API from SiteTwo: standby style, data sync, cutover, planned switchover and failback |
| [sitetwo-site.md](sitetwo-site.md) | The Minnesota property: unheated buildings, enclosure and heater, power, monitoring, UniFi network gear and shopping list |
| [tablo.md](tablo.md) | The Tablo DVR at SiteTwo and how family watches it remotely without opening ports |
| [CLAUDE.md](CLAUDE.md) | Guidance for editing these docs: who owns which facts, naming, and settled decisions |

Start with multiproject.md. Read the SiteTwo files only if you care about the standby site or the Tablo.

## Status

Design stage. The build order is in [multiproject.md](multiproject.md#8-build-order). Open questions (cold ratings of the hardware, the Tablo model and antenna signal, which devices family will watch on) are listed in [sitetwo-site.md](sitetwo-site.md#open-questions) and [tablo.md](tablo.md#open-questions).
