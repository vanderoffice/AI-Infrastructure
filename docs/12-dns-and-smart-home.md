# Network Services

## Why This Exists

Every device on your network makes DNS queries -- hundreds per hour. By default, those queries go to your ISP or to Google or Cloudflare. That means your ISP sees every domain you visit, ads load on every page, and trackers follow you across the web.

Network-wide DNS filtering fixes all of that at once. Instead of installing an ad blocker on every phone, laptop, tablet, and smart TV individually, you point the entire network at a DNS server you control. That server blocks ads, trackers, and malware domains before the connection is ever made. Every device benefits automatically -- even devices that do not support ad blockers (smart TVs, IoT sensors, game consoles).

Running two Pi-holes instead of one gives you redundancy. If the primary DNS server goes down for maintenance or crashes, the secondary picks up immediately. No one on the network notices. The smart home stack ties into the same server infrastructure, using MQTT messaging and Zigbee radios to control physical devices from software.

---

## DNS Architecture: Dual Pi-hole + Unbound

The DNS setup uses two Raspberry Pis running Pi-hole, with the primary configured for full recursive DNS resolution via Unbound and the secondary configured as a simpler Cloudflare-backed fallback. This is a deliberate asymmetry -- the primary is the performance tier, and the secondary is the reliability tier.

```
Client device (any)
  |
  |--> NetSentry (192.168.0.113) -- Primary
  |      Pi-hole --> Unbound --> Root nameservers (no third party)
  |
  |--> AlertNode (192.168.0.112) -- Secondary
         Pi-hole --> Cloudflare (1.1.1.1)
```

The router's DHCP server pushes both Pi-hole IPs to every client. Clients try the primary first and fall back to the secondary automatically.

---

## NetSentry -- Primary DNS (Pi 5)

NetSentry is the workhorse. It runs Pi-hole for filtering and Unbound for recursive resolution, which means DNS queries never leave your network to reach a third-party resolver.

| Property | Value |
|----------|-------|
| Hardware | Raspberry Pi 5 |
| IP Address | 192.168.0.113 |
| OS | Raspberry Pi OS (64-bit) |
| Pi-hole | v6.3 (Core), FTL v6.4.1 |
| Upstream DNS | Unbound on localhost (127.0.0.1#5335) |
| Web UI | HTTPS on port 443 (TLS enabled) |
| Temperature | ~57C (healthy, no throttling) |
| Uptime | 70+ days |

### What Unbound Does

Most DNS setups forward queries to Cloudflare (1.1.1.1) or Google (8.8.8.8). That is fast and simple, but it means a third party sees every domain you resolve. Unbound replaces that entire dependency by performing recursive resolution directly against the root nameservers.

When you request `example.com`, Unbound does not ask Cloudflare. It starts at the DNS root (`.`), asks which servers handle `.com`, then asks those servers which server handles `example.com`, and gets the answer directly from the authoritative nameserver. No middleman.

**Unbound configuration highlights:**

| Setting | Value | Why |
|---------|-------|-----|
| Threads | 4 | Matches Pi 5 core count |
| rrset-cache-size | 256MB | Large cache reduces repeat queries to root servers |
| msg-cache-size | 128MB | Generous for a Pi 5 with 4-8GB RAM |
| DNSSEC | Enabled | Validates DNS responses are not forged |
| QNAME minimization | Enabled | Sends minimal query info to each nameserver (privacy) |
| Prefetch | Enabled | Refreshes popular entries before they expire |
| Listening port | 5335 | Localhost only -- Pi-hole connects here |

The Unbound configuration file lives at `/etc/unbound/unbound.conf.d/pi-hole.conf` on the Pi.

### Other Services on NetSentry

NetSentry also hosts two monitoring dashboards:

| Service | Port | Purpose |
|---------|------|---------|
| Homepage | 3000 | Dashboard aggregating status of all services |
| Uptime Kuma | 3001 | Uptime monitoring for AlertNode and other machines |

---

## AlertNode -- Secondary DNS (Pi 3B+)

AlertNode is the fallback DNS server and the alerting hub. Its DNS configuration is intentionally simpler -- it forwards to Cloudflare instead of running its own recursive resolver. This means if NetSentry goes down (or if Unbound has an issue), DNS still works for the entire network.

| Property | Value |
|----------|-------|
| Hardware | Raspberry Pi 3B+ |
| IP Address | 192.168.0.112 |
| OS | Raspbian (32-bit, armv7l) |
| Pi-hole | v6.3 (Core), FTL v6.4.1 |
| Upstream DNS | 1.1.1.1 (Cloudflare) |
| Web UI | HTTPS on port 443 (TLS enabled) |
| Temperature | ~64C (thermal throttling active) |

**Note on the 32-bit OS:** The Pi 3B+ is an older board that shipped with 32-bit Raspbian. It works fine for DNS and lightweight services, but keep in mind that some modern packages may not have armv7l builds.

**Note on temperature:** The Pi 3B+ runs hot at 64C and is actively thermal throttling. This does not meaningfully impact DNS performance (the workload is trivial), but if you are reproducing this setup, add a heatsink or a small fan to the Pi 3B+. The Pi 5 handles its workload at 57C without issue because it ships with better cooling.

### Other Services on AlertNode

| Service | Port | Purpose |
|---------|------|---------|
| ntfy | 8080 | Push notification server for infrastructure alerts |
| Watchdog script | -- | Monitors NetSentry health every 5 minutes |

The ntfy server is the alerting backbone for the entire infrastructure. When something goes wrong -- a backup fails, a container crashes, a service goes down -- the alert lands on ntfy as a push notification to your phone.

---

## Gravity-Sync -- Blocklist Replication

Pi-hole maintains a gravity database of blocked domains. When you add a custom blocklist or whitelist an entry on the primary, you want that change reflected on the secondary. Gravity-sync handles this automatically.

| Property | Value |
|----------|-------|
| Direction | AlertNode pulls FROM NetSentry |
| Source of truth | NetSentry (primary) |
| Schedule | Hourly (systemd timer) |
| Mode | SMART (only syncs when gravity database has changed) |
| Auth | SSH key authentication between the two Pis |

**How it works:** A systemd timer on AlertNode fires every hour. The gravity-sync script checks whether NetSentry's gravity database has changed since the last sync. If it has, it pulls the updated database. If nothing changed, it does nothing. This keeps bandwidth and CPU usage minimal.

**Important:** You only manage blocklists on NetSentry. AlertNode automatically mirrors whatever NetSentry has. Do not add blocklists to AlertNode directly -- they will be overwritten on the next sync.

---

## Mutual Monitoring

The two Pis watch each other. If either goes down, the other detects it and sends an alert.

### NetSentry Monitors AlertNode

NetSentry runs Uptime Kuma (port 3001), which checks AlertNode's availability at regular intervals. If AlertNode stops responding, Uptime Kuma sends an alert via its configured notification channels.

### AlertNode Monitors NetSentry

AlertNode runs a custom watchdog script that performs three checks against NetSentry every 5 minutes:

| Check | Method | What It Proves |
|-------|--------|----------------|
| Ping test | 3 pings, 5s timeout | Network reachability |
| Pi-hole API | HTTP GET to /admin/api.php | Pi-hole is running and responding |
| DNS resolution | `dig @192.168.0.113` | Actual DNS queries are being answered |

The watchdog uses state tracking to prevent alert spam. If NetSentry goes down, you get one alert. You do not get an alert every 5 minutes until it comes back. When it recovers, you get a single recovery notification.

Alerts go to the ntfy topic `infrastructure-alerts`, which pushes to your phone.

---

## Local DNS Records

Pi-hole can serve as a local DNS resolver for internal services, so you can reach them by name instead of IP address.

| Hostname | Resolves To | Service |
|----------|-------------|---------|
| vault.vanderdev.local | 192.168.0.100 | Vaultwarden (password manager on ServerM2P) |
| n8n.local | 192.168.0.100 | n8n workflow engine (on ServerM2P) |

These records are configured in Pi-hole's Local DNS settings. They only work for devices using the Pi-holes as their DNS server (which is everything on the network, thanks to DHCP).

**Why `.local`?** Using `.local` keeps these names completely internal. They will never resolve on the public internet, and there is no risk of collision with real domain names. The tradeoff is that mDNS (Bonjour) also uses `.local`, which can occasionally cause resolution conflicts on macOS. Always have the raw IP address as a fallback.

---

## DHCP Integration

The Omada router's DHCP server pushes both Pi-hole IPs to every client that joins the network:

| Setting | Value |
|---------|-------|
| Primary DNS | 192.168.0.113 (NetSentry) |
| Secondary DNS | 192.168.0.112 (AlertNode) |

This is configured in the Omada controller's DHCP settings, not on the Pi-holes themselves. The Pi-holes do not run DHCP -- they only handle DNS. The router handles IP address assignment.

**Why this matters:** Clients do not need any manual configuration. A new laptop, a guest phone, a smart TV -- anything that connects to the network automatically uses the Pi-holes for DNS. No per-device setup required.

---

## Smart Home Stack

The smart home automation runs on ServerM2P alongside the other infrastructure services. It uses three Docker containers that work together: Home Assistant as the brain, Zigbee2MQTT as the radio bridge, and Mosquitto as the message bus.

```
Physical devices (lights, sensors, switches)
  |
  Zigbee radio (SLZB-MR1 coordinator)
  |
  Zigbee2MQTT (translates Zigbee --> MQTT messages)
  |
  Mosquitto (MQTT broker -- routes messages)
  |
  Home Assistant (automation engine, dashboards, integrations)
```

### Home Assistant

| Property | Value |
|----------|-------|
| Container | homeassistant |
| Port | 8123 |
| Host | ServerM2P (192.168.0.100) |
| Integrations | Zigbee2MQTT, Google Home, Apple HomeKit |

Home Assistant is the central hub. It provides a web dashboard for controlling devices, automations that trigger actions based on conditions (time, sensor readings, device states), and integrations with external platforms like Google Home and Apple HomeKit. If you want your Zigbee light to turn on at sunset, Home Assistant is where that rule lives.

### Zigbee2MQTT

| Property | Value |
|----------|-------|
| Container | zigbee2mqtt |
| Port | 8080 |
| Coordinator | SLZB-MR1 CC2652P7 |
| Coordinator address | tcp://192.168.0.21:7638 |
| socat proxy | ServerM2P port 7639 --> 192.168.0.21:7638 |

Zigbee2MQTT bridges the gap between Zigbee radio devices and the MQTT messaging protocol that Home Assistant understands. The SLZB-MR1 is an Ethernet-connected Zigbee coordinator -- it plugs into your network with an Ethernet cable instead of USB. This is a significant advantage because USB Zigbee sticks can have range and interference issues, and they tie the coordinator to a specific machine's USB port.

The socat proxy on ServerM2P (port 7639) bridges the TCP connection to the coordinator's native port (7638 on 192.168.0.21). This allows the Docker container to reach the coordinator through a stable local port.

**Supported devices:** The CC2652P7 chipset supports Zigbee 3.0, which covers the vast majority of modern Zigbee devices -- lights, motion sensors, door/window sensors, temperature sensors, smart plugs, and switches from brands like IKEA, Aqara, Sonoff, and Philips Hue.

### Mosquitto MQTT Broker

| Property | Value |
|----------|-------|
| Container | mosquitto |
| TCP port | 1883 |
| WebSocket port | 9001 |

Mosquitto is the message bus. Zigbee2MQTT publishes device state changes to MQTT topics, and Home Assistant subscribes to those topics. When a motion sensor detects movement, Zigbee2MQTT publishes a message. Home Assistant receives it and runs any automations tied to that sensor. The communication is bidirectional -- Home Assistant can also publish commands (turn on a light, set a thermostat) that Zigbee2MQTT picks up and sends to the physical device.

### zigbee-control.py

A utility script for direct Zigbee device control from the command line. Useful for testing and debugging without opening the Home Assistant UI. It publishes MQTT messages directly to Mosquitto, bypassing Home Assistant entirely.

---

## Reproduction Steps

### Step 1: Flash Two Raspberry Pis

Flash Raspberry Pi OS to SD cards for both Pis. Use the 64-bit version for a Pi 4 or Pi 5. If you are using a Pi 3B+, the 32-bit (armv7l) version is your only option.

Use the Raspberry Pi Imager to set hostname, enable SSH, and configure Wi-Fi during flashing. Assign static IPs either on the Pis themselves or via DHCP reservations on your router.

### Step 2: Install Pi-hole on Both

```bash
curl -sSL https://install.pi-hole.net | bash
```

Follow the interactive installer. Choose your upstream DNS (it does not matter yet -- you will change it after Unbound is configured on the primary). Note the admin password the installer generates.

### Step 3: Install Unbound on the Primary

```bash
sudo apt install unbound
```

Create the configuration file at `/etc/unbound/unbound.conf.d/pi-hole.conf`. The Pi-hole documentation has a complete example configuration for this integration. Key settings:

- Listen on `127.0.0.1` port `5335`
- Enable DNSSEC validation
- Set cache sizes appropriate for your Pi's RAM
- Enable QNAME minimization and prefetch

Restart Unbound and test:

```bash
sudo systemctl restart unbound
dig @127.0.0.1 -p 5335 example.com
```

### Step 4: Configure Pi-hole Upstreams

**Primary (NetSentry):** Set the upstream DNS to `127.0.0.1#5335` (Unbound). Remove all other upstreams.

**Secondary (AlertNode):** Set the upstream DNS to `1.1.1.1` (Cloudflare) or `9.9.9.9` (Quad9). This is your fallback -- keep it simple.

### Step 5: Install Gravity-Sync on the Secondary

Gravity-sync runs on the secondary and pulls from the primary. Follow the gravity-sync installation guide on GitHub. You will need:

- SSH key authentication from AlertNode to NetSentry (generate keys, copy the public key)
- The IP address of the primary Pi-hole
- A systemd timer set to hourly

### Step 6: Set Up the Watchdog

On the secondary, create a script that runs every 5 minutes (via cron or a systemd timer) and checks the primary's health with ping, HTTP, and DNS tests. Send alerts to ntfy or your notification service of choice.

### Step 7: Configure Router DHCP

In your router's DHCP settings, set the primary and secondary DNS servers to your two Pi-hole IPs. Every device on the network will receive these DNS servers automatically on its next DHCP lease renewal.

### Step 8: Smart Home (Optional)

If you want smart home automation, install Docker on your server and run:

- **Home Assistant** -- the automation engine
- **Zigbee2MQTT** -- the Zigbee radio bridge (requires a Zigbee coordinator like the SLZB-MR1 or a USB stick)
- **Mosquitto** -- the MQTT message broker

All three are available as official Docker images. A single `docker-compose.yml` can bring up the entire stack.

---

## Gotchas and Common Issues

**Pi 3B+ thermal throttling.** The Pi 3B+ runs hot and will thermal throttle at 60-65C. This does not break DNS (the workload is too light to be impacted), but it shortens the board's lifespan. A $5 heatsink-and-fan kit solves it. The Pi 5 has adequate built-in cooling and does not need extras for this workload.

**Gravity-sync SSH keys.** Gravity-sync uses SSH to pull the database from the primary. You must generate an SSH key pair on AlertNode and copy the public key to NetSentry's `~/.ssh/authorized_keys`. If gravity-sync fails silently, check SSH connectivity first: `ssh pi@192.168.0.113` from AlertNode.

**Pi-hole v6 API changes.** Pi-hole v6 introduced a new API that is not backward-compatible with v5 scripts. If you are adapting monitoring scripts or dashboards from older guides, check whether they use the v5 API (`/admin/api.php`) or the v6 API. The watchdog script in this setup uses the v5-compatible endpoint, which still works in v6.

**mDNS and `.local` conflicts.** macOS uses `.local` for Bonjour/mDNS discovery, which can conflict with Pi-hole's local DNS records that also use `.local`. If a `.local` hostname is not resolving correctly, try flushing the DNS cache (`sudo dscacheutil -flushcache` on macOS) or use the IP address directly.

**SLZB-MR1 is Ethernet, not USB.** The Zigbee coordinator connects to your network via Ethernet, not USB. This means it needs its own IP address and network access. It also means you can place it far from your server for better Zigbee range -- it just needs an Ethernet run. The socat proxy bridges the Docker container to the coordinator's network address.

**Zigbee mesh range.** Zigbee is a mesh protocol. Mains-powered devices (smart plugs, always-on lights) act as repeaters and extend the mesh. Battery-powered devices (sensors) do not repeat. If you have range issues, add a smart plug between the coordinator and the distant device -- it will relay traffic.
