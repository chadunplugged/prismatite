# Tablo at SiteTwo

## Goal

A Tablo (over-the-air DVR) is hooked up at SiteTwo and can be watched remotely by family. **Remote access must work without opening ports to the internet.**

This is independent of the API failover ([sitetwo-failover.md](sitetwo-failover.md)). The site it lives in (cold, power, network) is described in [sitetwo-site.md](sitetwo-site.md).

## Open questions

- Cold rating of the Tablo (datasheet). Add it to the cold-ratings check in [sitetwo-site.md](sitetwo-site.md#open-questions).
- There is an older (legacy) unit, model not remembered. A new one will probably be bought. Where will the antenna go, and is there an over-the-air signal at the site (check rabbitears.info or antennaweb.org)?
- Remote viewing: the viewers are family (you, parents, sister, others). Which devices (phones, TV streaming boxes, laptops)? Live TV, recordings, or both?

## Tablo setup
- **Network:** wired Ethernet to the UniFi switch, not Wi-Fi. Give it a DHCP reservation so its address never changes.
- **Antenna:** needs a coax run to an OTA antenna, ideally outdoors or in the attic, as high as possible. Antennas and coax handle the cold fine. Add a surge/grounding block on the coax, as for the fiber converters. An amplified antenna needs power for its preamp, so keep that inside the warm space.
- **Storage:** the Tablo records to a USB drive (or a built-in drive on older models). **Spinning drives are out**, so use a USB SSD. Check the Tablo's supported drive list and size limit. Recordings are not critical, so no backup is needed.
- **Cold:** the Tablo is a consumer device. Keep it in the heated enclosure, or somewhere that stays above its rated minimum. A coax pass-through into the enclosure is enough.
- **Heat and power:** it is low power (a few watts) and runs all the time. Put it on the always-on outlet, not on a Wyze plug, but a plug is fine if you want a remote power-cycle (it recovers by itself after power returns).

## Remote access options
| Option | How it works | Fit |
|---|---|---|
| **Tablo's built-in remote streaming** | The Tablo app connects through Tablo's cloud. Nothing to set up on the network. | Easiest. Check which models and plans support it and whether it needs a subscription. Works even if the NUC is off. |
| **Tailscale on the NUC as a subnet router** | The NUC advertises the SiteTwo LAN. Remote devices with Tailscale reach the Tablo as if local. | Reuses the Tailscale already planned for the API link. **The NUC would have to stay on all the time**, which is already recommended. Apps that find the Tablo by local broadcast may not discover it across a routed link. Test this before relying on it. |
| **UniFi gateway VPN (WireGuard or Teleport)** | The gateway itself is the VPN server. | Does not depend on the NUC. Good as a second way in, and for reaching the UniFi controller and other gear. |
| Cloudflare Tunnel | Video streaming through the tunnel is against Cloudflare's terms for the free plan. | **Do not use** for Tablo video. |

**Family viewers change the picture.** Parents and a sister will not want to install and manage a VPN. A VPN-only setup makes each of them a support call. So the family-facing path should be **the Tablo app's own remote streaming**, which needs nothing on their side beyond the app. This is the main reason to **buy a current-generation Tablo**: legacy units may not support remote streaming, or support it with older apps. Check which models support it before buying. The VPN is then for you (and the caretaker), as the admin path.

**If the NUC goes off:** the Tablo keeps recording on its own, and the Tablo app's built-in remote streaming is unaffected, because neither runs through the NUC. What stops is the Tailscale path into the site (the Tablo over VPN, the NUC itself, and the SiteTwo LAN from outside), plus the NUC's standby jobs (see [sitetwo-failover.md](sitetwo-failover.md)). The UniFi gateway VPN still works. So with the plan below, a dead NUC costs you convenience, not family viewing.

**Suggestion:** try Tablo's built-in remote streaming first. Add Tailscale on the NUC as the general-purpose way into the site (Tablo, UniFi, the NUC itself), and set up the UniFi gateway VPN as a backup path that works when the NUC is down. Do not forward ports on the gateway to the Tablo.

## Things to check
- **Upload speed:** a remote stream needs roughly 5–10 Mbps up per viewer for an HD OTA stream. The SiteTwo line is 1 Gbps symmetric ([sitetwo-site.md](sitetwo-site.md#known-facts)), so this is not a concern, even with several viewers plus the API and backups.
- **Tablo remote streaming** may transcode, which the Tablo handles in hardware. Several remote viewers at once may be more than it can do.
- **Alerts:** add "Tablo unreachable" to the uptime alerts ([sitetwo-site.md](sitetwo-site.md#remote-monitoring)).
- **Access list:** put the Tablo VPN on named devices only (Tailscale ACLs), so a lost phone does not give access to the whole site.

## Shopping list

| Item | Purpose | Notes |
|---|---|---|
| Tablo (model TBD) | Over-the-air DVR, watchable remotely | In the heated enclosure |
| OTA antenna, coax, surge/grounding block | Signal | Antenna high and outdoors or in the attic |
| USB SSD | Recording storage | No spinning drive; check the Tablo's supported drives |

## Order of work

1. Check the Tablo questions above: cold rating, over-the-air signal at the site, which model to buy.
2. This comes after the network and enclosure are in place ([sitetwo-site.md](sitetwo-site.md#4-order-of-work), steps 2–3).
3. Hook up the Tablo (antenna, Ethernet, SSD). Test remote viewing with the built-in option first, then over Tailscale.
4. Add the UniFi gateway VPN as the backup path, and Tailscale ACLs for named devices.
