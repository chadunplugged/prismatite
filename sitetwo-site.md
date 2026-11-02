# SiteTwo: the Minnesota property

## Goal

Make the unused fiber connection at the Minnesota property a dependable place to keep gear: a small server that can host the home API ([sitetwo-failover.md](sitetwo-failover.md)), working Wi-Fi in both buildings, and the Tablo ([tablo.md](tablo.md)).

This file covers the building, the cold, power, monitoring and network gear. It does not cover API failover or the Tablo.

## Known facts

- **Not critical.** If SiteTwo is offline, that is not a big deal. It favors simple and cheap over redundant and resilient.
- **Unheated, but very well insulated.** Minnesota winters reach -30°F (-34°C) and colder. No spot in the building stays above freezing. It takes a long time to get cold and a long time to warm up, but expect it to stay below 0°C for a month or two each winter.
- **Power is quite reliable.**
- **Internet is better than at Home.** The SiteTwo line is a symmetric 1 Gbps fiber line (1 Gbps up and down). It is about twice the speed of the home line and more stable. The home line is a business-class line that is not 100% stable, for a variety of reasons. Other files link here for these numbers.
- **A caretaker lives nearby** and can typically check or adjust things within a day.
- **Two buildings**, joined by a fiber link of about 1000 ft (about 305 m), with a CAT-to-fiber media converter at each end. Both need Wi-Fi.
- **Temperature sensing:** a wired USB temperature sensor goes on the NUC and on whatever server is brought out to the site.
- **Remote power control via Wyze smart plugs** (server power, and possibly the enclosure panels).
- **Existing hardware on site:** an ASUS NUC with a dual-core Intel Celeron N4000 @ 1.10 GHz, 8 GB RAM and 100 GB of disk. It can always be on. It has not needed to be, so it is usually off. It is turned on through a Wyze plug, mostly to read temperatures with the `sensors` command. The routers and other network gear are always on.
- Spinning drives are out.

## Open questions

- Cold ratings: check the datasheets of the Wyze plug, the NUC, the Pi, the building 2 media converter and the UniFi gear for operating temperature (see section 1). The Tablo's rating is tracked in [tablo.md](tablo.md).

## 1. The server and the cold

### What to run
The SiteTwo box holds backups and, during an outage, runs the API and Postgres for a few low-traffic sites ([sitetwo-failover.md](sitetwo-failover.md)). That is a light job, so a small machine is enough:

| Option | Fit | Notes |
|---|---|---|
| **Existing NUC** (Celeron N4000, 8 GB, 100 GB) | Good start. Already there, costs nothing, and can run all the time. | Weak CPU but enough for low traffic. The file set is under 1 GB, so 100 GB is plenty for the OS and backup generations. |
| **Raspberry Pi 5** (or similar) | Fine if the NUC is busy or you want a dedicated box. | Use an NVMe SSD (via the PCIe HAT), not an SD card. Low power. 8 GB RAM version. |
| **Xeon server** | Works, but overkill for the load. | See the heated-enclosure idea below. Only worth it if you want the heat. |

**Recommendation:** start with the NUC. It proves the whole setup for free. Move to a Pi 5 or a bigger machine only if the NUC is too slow or too full.

### The real cold risks
- **Starting cold.** Most consumer hardware is rated to *run* at about 0–40°C. Booting well below that is outside spec.
- **Condensation.** Warm, moist air meets cold boards when a door opens or a cold machine warms up. This is the main killer, more than the cold itself.
- **Batteries.** Lead-acid UPS batteries lose much of their capacity in the cold. Lithium (LiFePO4) batteries **must not be charged below 0°C**.
- **Fans and lubricants** stiffen in the cold. Fanless (a Pi) avoids this. The NUC has a small fan.

### Approach: insulated enclosure that keeps the gear warm
Because the work is not critical and a caretaker is close, **a consumer-grade machine in a warm box is enough.** Wide-temperature industrial PCs and industrial SSDs (Advantech, Neousys, OnLogic Karbon, Innodisk) would also work, but cost far more than this needs. They are still a good option if you would rather not build an enclosure.

The enclosure options:

1. **Insulated box with a thermostat heater.** An insulated NEMA-style cabinet with a **thermostat-controlled enclosure heater**, set to about 5°C (40°F), plus desiccant packs. Once running, the computer, gateway and switch put out heat that stays in the box. The heater mainly covers cold starts and power outages. Put the gateway, switch and server **all in this box**, so everything shares one warm, dry space.
2. **Server-as-heater (Xeon idea).** An enclosure that closes its panels in the cold and uses the machine's own heat as the heated space, then opens them when it gets warm. Notes:
   - A Xeon rack server draws maybe 100–200 W. That is a decent heater, but also a constant power bill all year, and the fans are loud and move a lot of air.
   - You need a **summer plan**. Panels must open (or vent) when it is warm, or the box will cook the server.
   - If the panel opening/closing runs off a Wyze plug, remember a plug is only on/off. Use a **linear actuator with built-in limit switches** (it stops itself at each end), and decide what the safe state is if Wi-Fi or the plug fails. A passive fallback (a wax-cylinder thermal vent opener, which needs no power) is more dependable than a Wi-Fi-controlled panel.
   - It only makes sense if a Xeon is wanted anyway. For this load, option 1 with the NUC or a Pi is simpler.

### Power
- Power is quite reliable, so a **UPS is optional.** In winter a lead-acid UPS may lose much of its capacity, and a lithium UPS can't charge when cold, so it may be a losing battle. Reasonable choice: **skip the UPS and accept the risk**, since downtime is not a big deal. A UPS *inside* the heated enclosure is the only version that makes sense, and only if it ever becomes worth it.
- BIOS setting: **power on after AC loss**, so the machine comes back by itself after an outage. (The Pi 5 powers on at once when power returns.)
- **Wyze smart plugs for remote power control.** Good enough for power-cycling a hung server, and they are cheap. Points to check:
  - The plug needs **Wi-Fi to work.** Keep the gateway and AP on a separate, always-on outlet, never on a plug that the Wyze plug controls, or a lockup could leave you unable to reach the plug.
  - Confirm the **cold rating** of the plug on its datasheet. Wyze plugs are consumer devices and are not guaranteed to work well at -30°F. Keep them inside the heated enclosure.
  - Hard power-cycling is rough on filesystems. Use a journaling filesystem (ext4/ZFS) and try a soft reboot over Tailscale first.
  - Because of the caretaker, the Wyze plug is a convenience, not the only way back in.
- **Heater interlock:** after a long outage, the box can be colder than the computer's rated range. Delay power to the computer until the enclosure warms up, using a thermostat relay (best) or a Wyze schedule/rule (simpler, less exact).

### Remote monitoring
- Wired USB temperature sensors, one on the NUC and one on the server, placed in the enclosure and sent to the same uptime alerting as the API.
- The NUC can stay on, so its sensor can report continuously. Still put one sensor on something that stays powered, such as the gateway, so the "enclosure below 0°C" alert works even if the NUC is off.
- Alerts on: enclosure below 0°C, disk nearly full, UPS on battery (if ever added). Backup-age and tunnel alerts are in [sitetwo-failover.md](sitetwo-failover.md#3-alerts-that-belong-to-failover). The Tablo alert is in [tablo.md](tablo.md).

## 2. Network gear

### Gateway: a UniFi Dream Machine is a fit, with a caveat

A UniFi gateway at SiteTwo makes sense: one dashboard for both buildings, and remote management from the UniFi app.

- **Caveat:** UniFi gateways are indoor products. Their rated operating range starts around -5 to 0°C. Put the gateway **in the heated enclosure**, not on an open shelf.
- Model: a **UDM Pro or UDM SE** if you want a rack-mount unit with NVR and room to grow. A **Cloud Gateway Ultra or Max** is smaller and cheaper, and enough for one fiber line and two buildings. A small one is easier to fit in an enclosure.

### Building 2: access points, not routers

Don't put "piggyback routers" in the far building. Each router makes its own network, so you get double NAT, devices drop when moving between buildings, and you manage two systems.

Instead, extend the **same network**:
- The fiber link ends in a media converter in building 2, which feeds a **managed PoE switch** there.
- **UniFi access points** run off that switch. They are managed by the same gateway, use the same Wi-Fi name, and devices roam between buildings.

### Use outdoor-rated APs and switches indoors
Outdoor UniFi gear is rated for cold (roughly -30 to -40°C), unlike the indoor models. In unheated buildings, use it inside:
- **APs:** U6 Mesh, U7 Outdoor, or U6 Mesh Pro. Mount them indoors.
- **Switch in building 2:** USW Flex (outdoor-rated, powered by PoE) if a few ports are enough. Otherwise put an indoor switch in a small heated enclosure.
- Check each datasheet's operating range before buying.

### The link between buildings
The existing link is fiber. The fiber from the road terminates in building 1, then a CAT-to-fiber media converter carries the signal about 1000 ft (about 305 m) to building 2. Fiber has no electrical connection between the buildings, so the copper concerns (ground differences, lightning surges, the 100 m limit) do not apply.
- The media converter in **building 2** sits in the cold. Use a cold-rated model, or put it in the heated enclosure with the switch. It needs power, so plan for an outlet there.
- Put surge protection on the power feeds of both converters.

### Link to Home
- The server-to-server Tailscale or WireGuard link the API needs is described in [sitetwo-failover.md](sitetwo-failover.md#network-between-sites).
- If Home also gets a UniFi gateway later, **UniFi Site Magic** can join both sites' networks. Not needed for the API.

## 3. Rough shopping list

Tablo items are in [tablo.md](tablo.md#shopping-list).

| Item | Purpose | Notes |
|---|---|---|
| ASUS NUC (already on site) | Standby API + Postgres + backup target | Check disk space; add a bigger SSD if needed |
| *Optional:* Raspberry Pi 5 8 GB + NVMe | Dedicated standby if the NUC is busy | NVMe, not SD card |
| UniFi Cloud Gateway (Ultra/Max) or UDM Pro/SE | Router, firewall, UniFi controller | Inside the heated enclosure |
| Insulated enclosure + thermostat heater + desiccant | Warm, dry space for the gear | Size for the gateway, switch, server and Tablo |
| Wyze plugs (2–3) | Remote power-cycle of the server; optional enclosure panel actuator | Keep the gateway on its own outlet; check cold rating |
| Wired USB temperature sensor, one on the NUC and one on the server | Temperature alerts | Humidity too if the model reports it. Reads only while its host is on, so keep the NUC on |
| Outdoor-rated UniFi APs (one or more per building) | Wi-Fi in both buildings | U6 Mesh, U7 Outdoor |
| USW Flex or a managed PoE switch (building 2) | Feeds the APs in building 2 | |
| Media converter for building 2 (link already in place) | Turns the ~1000 ft fiber run into Ethernet | Cold-rated, or inside the heated enclosure |
| *Optional:* small lead-acid UPS | Ride through short outages | Only inside the heated enclosure; probably skip |
| *Optional:* linear actuator with limit switches, or wax-cylinder vent | Panels for a server-as-heater enclosure | Only if a Xeon is used |

## 4. Order of work

1. Check the cold ratings listed under open questions.
2. Network first: gateway, enclosure, APs in both buildings. This is useful on its own.
3. Put the NUC (or Pi) in the enclosure on a Wyze plug. Tailscale to Home. Enable subnet routing so the SiteTwo LAN is reachable remotely.
4. Temperature and humidity alerts before winter.
5. Only if needed later: a UPS, a bigger server.

The Tablo has its own order of work in [tablo.md](tablo.md#order-of-work). API failover steps are in [sitetwo-failover.md](sitetwo-failover.md#4-order-of-work).
