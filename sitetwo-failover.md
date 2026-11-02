# SiteTwo failover: running the home API from Minnesota

## Goal

Use the SiteTwo box (Minnesota) as a second place to run the home API ([multiproject.md](multiproject.md)). If Home (the R730 and its line) goes down or gets unreliable, the API can be moved to SiteTwo, and back again afterwards.

This file covers only the API and data side. For the building, cold, power and network gear see [sitetwo-site.md](sitetwo-site.md). For the Tablo see [tablo.md](tablo.md).

## Facts that drive the design

- **Not critical.** None of the APIs being moved are critical. If SiteTwo is offline, that is not a big deal. This is the biggest design input: it favors simple and cheap over redundant and resilient.
- **Cutover does not need to be automatic.** Restoring the API's database from backup onto SiteTwo is acceptable. Simplicity is preferred over services that might break. Automatic cutover is fine only if it is easy.
- **A caretaker lives nearby** and can typically check or adjust things within a day.
- **Either site can be the active one.** The NUC at SiteTwo is as valid a host for the API and Postgres as the R730 is. Choose the active site by which connection is working better at the time, not by which machine is the main server. Line speeds and stability are in [sitetwo-site.md](sitetwo-site.md#known-facts).
- **The file set is small.** lab131's shared files are under 1 GB and expected to stay that small.

## 1. Standby style

The usual terms:

| Term | What it means | Switch time |
|---|---|---|
| **Cold standby** | A spare machine that is off or idle. Restore from backup when needed. | Hours |
| Warm standby | The spare runs all the time and receives live data, but serves no traffic. A person promotes it when needed. | Minutes |
| Hot standby / automatic failover | Traffic moves by itself when health checks fail. | Seconds |

**Recommendation: start with backup-and-restore (cold standby) and manual cutover.** Since downtime is acceptable, this is the simplest design with the fewest parts that can break:

- Home ships Postgres backups and file copies to SiteTwo on a timer.
- On a real outage, someone restores the latest backup, starts the API and repoints DNS. Expect to lose whatever changed since the last backup, so run backups often enough that this is tolerable (hourly, or WAL archiving, if it matters).
- The standby machine can be left running the whole time. It does not have to sit cold.

**Upgrade path:** if losing a few hours of data turns out to be a problem, add Postgres streaming replication later. It makes SiteTwo a live copy seconds behind, and promotion takes minutes. It adds a moving part (replication health, failback rebuilds), so only do it if needed.

**Automatic failover is not recommended.** With one database, a dropped link between sites while both are up can leave both thinking they are primary (a "split brain"). Manual cutover avoids that, and the caretaker or you can do it within a day anyway.

The direction can also switch: whichever site is primary, the other is the standby.

## 2. Design

```
                Cloudflare DNS: api.<domain>
                   │                    │
          tunnel "home"          tunnel "sitetwo"
                   │                    │
      R730 (primary)                SiteTwo box (standby)
      api + Postgres  ── Tailscale ──►  backups + files land here
                                        api (stopped) until cutover
```

The diagram shows the default direction. Swap the labels when SiteTwo is the active site.

### Network between sites
- A **Tailscale** (or WireGuard) link between the R730 and the SiteTwo box. Tailscale is simpler: no ports to open on either side, and it works behind NAT.
- Backup traffic runs over this link, never over the public internet.

### Data
- **Postgres backups** (`pg_dump` on a timer, or `pgBackRest`/WAL archiving for finer recovery points) copied to SiteTwo. Keep several generations, not just the latest.
- **Files** (lab131 shares, uploads): `rsync` on a timer, or ZFS snapshots with `zfs send` if both machines use ZFS. The file set is under 1 GB, so a timer-driven `rsync` is plenty.
- **Config and images**: the same `docker-compose.yml` and `tenants.yml` in a git repo, pulled at both sites.
- **Backup policy (3-2-1).** SiteTwo is the second backup location. The offsite cloud copy from [multiproject.md](multiproject.md#backups) stays too: two sites plus a cloud copy.
- **Test the restore** on the SiteTwo box every so often. A backup that has never been restored is a guess.

### Traffic
- Each site runs its own Cloudflare Tunnel (`home` and `sitetwo`). Both can stay connected, so you can see remotely that SiteTwo is alive. The single-tunnel setup at Home is described in [multiproject.md](multiproject.md#option-a-cloudflare-tunnel-recommended).
- The `api.<domain>` DNS records point at the active site's tunnel.
- Don't run one tunnel with connectors at both sites. Cloudflare would send traffic to both sites at once.
- The records to repoint are those of every tenant hosted by the home API (the list in `tenants.yml`).

### Cutover runbook: emergency (one script, run by hand)
1. Confirm that Home is really down, not just slow.
2. Restore the latest Postgres backup at SiteTwo. (If replication is added later: promote the replica instead, `pg_ctl promote`.)
3. Start the `api` container at SiteTwo.
4. Repoint the `api.<domain>` records to the `sitetwo` tunnel. Do it with the Cloudflare API or Terraform, so the script does every tenant domain at once.

### Planned switchover (Home is up but unreliable)
Used when the plan is to run from SiteTwo while the home line is flaky, rather than after a crash.
1. Pick a quiet window. Stop the API at Home so nothing is written.
2. Take a fresh dump at Home, copy it to SiteTwo and restore it. Sync the files.
3. Start the API at SiteTwo and repoint the DNS records, as in steps 3 and 4 above.
4. Leave Home stopped. It is now the standby and must not serve traffic.

### Failback
1. When Home is back, copy the current SiteTwo data back to Home (a fresh dump and restore, or `pg_basebackup` if replication is in use). Don't start the old home database as is. It is missing everything written while SiteTwo was primary.
2. During a quiet window, run the planned switchover in the other direction.

### Optional: automatic traffic switching
Cloudflare Load Balancing (about $5 a month) has health checks and can switch traffic automatically. It is easy to set up, but only the *traffic* switches. The database would still need restoring, so it is most useful for a "we are down" maintenance page. Not on day one.

## 3. Alerts that belong to failover

These go to the same uptime alerting as the API (hardware and enclosure alerts are in [sitetwo-site.md](sitetwo-site.md#remote-monitoring)):
- Backup too old.
- The `sitetwo` tunnel down.
- Replica lag, if replication is ever added.

Since the caretaker can reach the site within a day, alert text should say what to check.

## 4. Order of work

1. SiteTwo box in place and on Tailscale ([sitetwo-site.md](sitetwo-site.md) steps 2–3).
2. Backups and file sync from Home to SiteTwo. Then use it as the second backup location.
3. Second Cloudflare Tunnel and the cutover script. **Practice a restore and a cutover in summer**, before the first cold season, and write down the steps for the caretaker too.
4. Backup-age and tunnel alerts before winter.
5. Only if needed later: streaming replication, automatic traffic switching.
