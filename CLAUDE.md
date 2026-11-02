# Prismatite: home infrastructure planning docs

This directory holds design docs (no code yet) for a self-hosted multi-tenant
Java API on a home server, a standby site in Minnesota, and related projects.
Docs are the source of truth. Edit them; don't duplicate their content here.

## Document map and ownership

| File | Owns | Does NOT contain |
|---|---|---|
| multiproject.md | App design: tenants, modules, API, data, Cloudflare ingress, build order | Physical site, cold, networking gear, Tablo |
| sitetwo-failover.md | Standby/cutover/failback for the API, inter-site data sync | Enclosure, UniFi, Tablo |
| sitetwo-site.md | Minnesota facilities: enclosure, power, cold, network gear, monitoring, line speeds | API design, Tablo |
| tablo.md | Tablo DVR and remote viewing | API/failover logic |

## Rules

- **One fact, one home.** State a fact in the owning file only; elsewhere
  link to it. Current homes for shared facts:
  - line speeds/stability: sitetwo-site.md
  - "NUC and R730 are both valid active hosts" and 3-2-1 backups: sitetwo-failover.md
  - single-tunnel setup at Home: multiproject.md; dual-tunnel design: sitetwo-failover.md
- Other files refer to sites by role ("primary", "standby"), not hardware,
  except in the file that owns the hardware.
- Use one name per concept: **Home** (R730, home line) and **SiteTwo**
  (Minnesota NUC). Never "siteone".
- Cross-references use relative links like [sitetwo-failover.md](sitetwo-failover.md).
- When a tenant is added or removed, update in lockstep: the tenant table, the
  `tenants.yml` sample, the cloudflared ingress sample, and the failover
  cutover scope.

## Settled decisions (don't re-litigate without being asked)

- Astro sites stay on static hosting; only the API runs at home.
- Cloudflare Tunnel in front of everything; home IP never in DNS.
- One Spring Boot app, tenant picked by Host header; Postgres schema per tenant.
- Passwordless magic-link auth, invite-only.
- SiteTwo: cold standby, manual cutover, no automatic failover (split-brain).
- No spinning drives; no port-forwarding to the Tablo; no Tablo video via
  Cloudflare Tunnel (ToS).
- Downtime is acceptable: favor simple and cheap over redundant.

## Conventions for edits

- Keep the existing style: short sections, tables for option comparisons,
  an explicit recommendation per decision, YAML/shell sketches in fences.
- Mark unknowns under "Open questions" in the owning file; remove them
  when answered, and fix any "everything is answered" claims.
- Don't invent hardware specs or cold ratings; flag them as "check datasheet".
