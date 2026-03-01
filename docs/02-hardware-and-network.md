# Chapter 2: Hardware and Network

## Why This Exists

A single computer can only do so much. When you start running AI workloads, hosting services, managing DNS, monitoring uptime, and deploying to production, you quickly outgrow one machine. The solution is a fleet of purpose-built machines connected by a secure mesh network.

This chapter documents every machine in the infrastructure, what it does, and how they all talk to each other. The goal is a setup where you can type `ssh server` from anywhere in the world and land on the right machine, securely, without thinking about IP addresses or VPN configurations.

The general flow looks like this:

```
Dev (StudioM4 / OfficeM4P) --> Test (ServerM2P) --> Prod (VPS)
```

Code is written on development machines, tested on the home server, and deployed to a public VPS. Supporting infrastructure (DNS, monitoring, alerting) runs on dedicated low-power devices that stay on 24/7.

---

## The Machine Inventory

Seven machines, each with a distinct role. No single point of failure for critical services.

| Machine | Hostname | User | Hardware | LAN IP | Tailscale IP | Role | Always-On |
|---------|----------|------|----------|--------|--------------|------|-----------|
| StudioM4 | StudioM4.local | slate | Apple M4 | 192.168.0.102 | 100.68.91.96 | Primary Development | No |
| OfficeM4P | OfficeM4P.local | quartz | Apple M4 Pro, 24 GB | 192.168.0.101 | 100.102.76.65 | Heavy AI, Research | Yes |
| ServerM2P | ServerM2P.local | commandervander | Apple M2 Pro, headless | 192.168.0.100 | 100.74.27.128 | Home Server, MCP Hub, Test | Yes |
| VPS | srv1060880.hstgr.cloud | root | Hostinger Ubuntu 24.04 | 212.38.95.33 | 100.111.63.3 | Production | Yes |
| NetSentry | netsentry.local | pi | Raspberry Pi 5 | 192.168.0.113 | 100.72.66.9 | Primary DNS, Monitoring | Yes |
| AlertNode | alertnode.local | pi | Raspberry Pi 3B+ | 192.168.0.112 | 100.69.47.71 | Secondary DNS, Alerting | Yes |
| S23+ | samsung-sm-s916u | -- | Samsung Galaxy S23+ | -- | 100.113.201.9 | Mobile Dev, Media | No |

**What each machine does in plain English:**

- **StudioM4** -- The daily driver. Code editing, Claude Code sessions, media production. Sleeps when not in use.
- **OfficeM4P** -- The heavy lifter. 24 GB of unified memory makes it the go-to for local AI inference, GIS processing, and deep research. Stays on to serve as a secondary compute node.
- **ServerM2P** -- The backbone. Runs headless (no monitor) as a home server. Hosts MCP servers, databases, container orchestration, backups, and the home automation stack. Think of it as the central nervous system.
- **VPS** -- The public face. Hosts the production website, public APIs, and anything that needs to be reachable from the internet. Runs on a Hostinger VPS with Ubuntu 24.04.
- **NetSentry** -- Primary DNS server and network monitor. Runs Pi-hole with Unbound for recursive DNS resolution. Monitors the health of other machines.
- **AlertNode** -- Secondary DNS server and alerting hub. Runs Pi-hole with Cloudflare as upstream fallback. Also runs an ntfy push notification server for infrastructure alerts.
- **S23+** -- A phone, but also a development and media capture device. Connects to the mesh via Tailscale for remote access and ADB debugging.

---

## Network Hardware

The local network runs on a TP-Link Omada SDN (Software-Defined Networking) stack. This is enterprise-grade gear that gives you centralized management, VLANs, and proper firewall rules -- overkill for most homes, but exactly right for a development network.

| Equipment | Model | Function |
|-----------|-------|----------|
| Gateway/Router/Firewall | ER707-M2 | Multi-Gigabit routing, firewall rules, inter-VLAN routing |
| Hardware Controller | OC300 | Centralized management for all Omada devices |
| Wireless Access Point | AX6000 | Wi-Fi 6/6E for all wireless clients |

**Management:** The Omada Cloud Portal at `omada.tplinkcloud.com` provides remote management. Use the cloud portal, not the gateway's local web UI.

**Home WiFi SSID:** `AD6XN` (this matters for SSH smart routing -- covered later in this chapter).

**If you are scaling down:** You do not need enterprise networking gear. A consumer router works fine. The important pieces are static IP assignments (or DHCP reservations) for your servers and the ability to set custom DNS servers in DHCP settings.

---

## Network Zones

The network is divided into zones to isolate traffic by trust level.

| Zone | Subnet | Machines | Access Level |
|------|--------|----------|--------------|
| Trusted/Dev LAN | 192.168.0.x | StudioM4 (.102), OfficeM4P (.101), ServerM2P (.100) | Full inter-device SSH, key auth only |
| Tailscale Overlay | 100.x.x.x | All 7 devices | Encrypted WireGuard mesh, works from anywhere |
| IoT/Media VLAN | Separate VLAN | Smart home devices, media players | Isolated from Trusted zone. Cannot reach dev machines. |
| DMZ/Public | Hostinger datacenter | VPS (212.38.95.33) | Containerized, minimal attack surface, no inbound except 80/443 |

**Key principle:** Dev machines live on the trusted LAN and can reach everything. IoT devices are jailed in their own VLAN so a compromised smart bulb cannot SSH into your server. The VPS is completely separate -- it lives in a datacenter and only accepts traffic on ports 80 and 443.

---

## Why Tailscale?

If you have machines in multiple locations (home, office, datacenter, your pocket), you need a way for them to talk to each other securely. Traditionally, this means setting up a VPN server, opening ports on your router, and dealing with dynamic IP addresses. Tailscale eliminates all of that.

**What Tailscale actually does:**

- Creates a peer-to-peer encrypted mesh network using WireGuard (a modern, fast VPN protocol)
- Every device gets a stable `100.x.x.x` IP address that never changes, regardless of what physical network it is on
- Devices talk directly to each other (peer-to-peer) whenever possible, not through a central server
- No port forwarding required. No firewall rules to maintain. No VPN server to run.
- Works behind NAT, on cellular networks, at hotels, everywhere

**What this means in practice:**

```
# From your laptop at a coffee shop:
ssh commandervander@100.74.27.128    # Connects to your home server
curl http://100.72.66.9/admin        # Opens your Pi-hole dashboard

# Same commands work identically from your couch at home.
# The IP addresses never change.
```

**Getting started:**

1. Go to [tailscale.com](https://tailscale.com) and create an account (free tier supports up to 100 devices)
2. Install Tailscale on each device (macOS, Linux, Android, iOS -- all supported)
3. Log in on each device with the same account
4. Done. Every device can now reach every other device via its `100.x.x.x` address.

No configuration files. No certificates to manage. No ports to open.

---

## DNS Architecture

DNS (Domain Name System) translates human-readable names like `vault.vanderdev.local` into IP addresses like `192.168.0.100`. This infrastructure runs its own DNS servers instead of relying on your ISP or Google/Cloudflare.

### Why run your own DNS?

- **Ad blocking:** Pi-hole blocks ads, trackers, and telemetry at the network level. Every device on the network benefits without installing anything.
- **Privacy:** With Unbound as a recursive resolver, DNS queries go directly to root nameservers instead of being logged by Google (8.8.8.8) or Cloudflare (1.1.1.1).
- **Local names:** Custom DNS records let you use names like `vault.vanderdev.local` for internal services.

### The dual Pi-hole setup

Two DNS servers provide redundancy. If one goes down, the other takes over automatically.

| Priority | Server | IP | Upstream Resolver | Why |
|----------|--------|-----|-------------------|-----|
| Primary | NetSentry (Pi 5) | 192.168.0.113 | Unbound (recursive) | Queries root nameservers directly. Maximum privacy. No upstream dependency. |
| Secondary | AlertNode (Pi 3B+) | 192.168.0.112 | Cloudflare (1.1.1.1) | Fast fallback. If NetSentry is down, DNS still works via Cloudflare. |

### How the pieces fit together

```
Client device (laptop, phone, etc.)
    |
    v
Pi-hole (NetSentry or AlertNode)
    |-- Is it an ad/tracker domain? --> Block it. Return 0.0.0.0.
    |-- Is it a local DNS record?   --> Return the local IP.
    |-- Otherwise                   --> Forward to upstream resolver.
                                          |
                          NetSentry: Unbound (recursive, talks to root nameservers)
                          AlertNode: Cloudflare 1.1.1.1 (fallback)
```

### Blocklist sync

The two Pi-holes need identical blocklists. **gravity-sync** handles this:

- Runs on AlertNode via a systemd timer
- Replicates Pi-hole blocklists and configuration from NetSentry to AlertNode
- Runs on a regular schedule to keep both servers in sync
- If you add a blocklist on NetSentry, it automatically appears on AlertNode

### Mutual monitoring

Each Pi watches the other:

- **NetSentry** monitors AlertNode via Uptime Kuma (a self-hosted uptime monitor)
- **AlertNode** monitors NetSentry via a watchdog script that pings it and tests DNS resolution every 5 minutes
- If either server goes down, an alert fires via ntfy (push notifications)

### Local DNS records

Custom DNS entries are configured in Pi-hole's configuration file (`/etc/pihole/pihole.toml`):

| Name | Resolves To | Service |
|------|-------------|---------|
| `vault.vanderdev.local` | 192.168.0.100 | Vaultwarden (password manager) |
| `n8n.local` | 192.168.0.100 | n8n (workflow automation) |

### DHCP configuration

The Omada router pushes both Pi-hole IPs to every device on the network via DHCP. This means every device automatically uses Pi-hole for DNS without any manual configuration.

**Path:** Omada Controller --> Network Config --> LAN --> DHCP Server --> DNS Server: Custom --> `192.168.0.113, 192.168.0.112`

---

## SSH Smart Routing

Typing IP addresses every time you SSH into a machine is tedious and error-prone. The SSH config uses a two-layer system that lets you type `ssh server` and automatically connect to the right machine using the best available path.

### How it works

The SSH config (`~/.ssh/config`) has two layers:

1. **Match blocks** (top of file) -- Decide which IP address to use based on your current network
2. **Host blocks** (below) -- Define credentials (username, SSH key) for each machine

```
# Layer 1: Match blocks -- dynamic hostname resolution
# "Am I on the home network?" If yes, use .local (mDNS). If no, use Tailscale.

Match host server,serverm2p exec "~/dotfiles/scripts/is-home.sh"
    HostName ServerM2P.local          # Home: use mDNS (fast, local)

Match host server,serverm2p
    HostName 100.74.27.128            # Away: use Tailscale IP (encrypted mesh)

# Layer 2: Host blocks -- credentials only (no HostName here)
Host server serverm2p
    User commandervander
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

### The detection script: `is-home.sh`

This script determines if you are on the home network:

1. Pings the gateway at `192.168.0.1` -- if it responds, you are home
2. Checks the current WiFi SSID for `AD6XN` -- if it matches, you are home
3. If neither check passes, you are remote

The script must exit `0` (home) or `1` (remote). SSH uses the exit code to decide which Match block to use.

**Critical requirement:** The script must be fast (under 1 second) and executable (`chmod +x`).

### SSH alias table

| Alias(es) | User | Target (Home) | Target (Remote) |
|-----------|------|---------------|-----------------|
| server, serverm2p | commandervander | ServerM2P.local | 100.74.27.128 |
| office, officem4p | quartz | OfficeM4P.local | 100.102.76.65 |
| netsentry | pi | netsentry.local | 100.72.66.9 |
| alertnode | pi | alertnode.local | 100.69.47.71 |
| vps | root | 100.111.63.3 (Tailscale preferred) | 212.38.95.33 (public fallback) |
| github.com | git | -- | -- |

**Note on VPS:** The VPS prefers Tailscale even from home because it avoids exposing SSH on the public IP. The public IP (`212.38.95.33`) is a last-resort fallback.

### Security requirements

- **Ed25519 keys only.** No RSA, no passwords. Generate with: `ssh-keygen -t ed25519`
- **`IdentitiesOnly yes`** on every host. This prevents the SSH agent from trying every key it has loaded, which can trigger brute-force lockouts on strict servers.

---

## Reproduction Steps

You do not need all seven machines to benefit from this architecture. Start with two (a dev machine and a server) and add more as your needs grow.

### 1. Install Tailscale on every machine

```bash
# macOS
brew install tailscale

# Ubuntu/Debian
curl -fsSL https://tailscale.com/install.sh | sh

# Raspberry Pi (Raspbian)
curl -fsSL https://tailscale.com/install.sh | sh

# Android
# Install from Google Play Store
```

### 2. Authenticate each device

```bash
sudo tailscale up
# Follow the URL to authenticate. Use the same account for all devices.
```

After authentication, each device gets a stable `100.x.x.x` IP. Note these down.

### 3. Set up Pi-hole (optional but recommended)

If you have one or two Raspberry Pis:

```bash
# On the Pi:
curl -sSL https://install.pi-hole.net | bash
```

For the primary DNS server, also install Unbound for recursive resolution:

```bash
sudo apt install unbound
```

Configure Pi-hole to use Unbound as its upstream DNS server (127.0.0.1 port 5335).

If you do not have Pis, skip this step and use Cloudflare (1.1.1.1) or Quad9 (9.9.9.9) as your DNS.

### 4. Configure DHCP to push DNS servers

In your router's DHCP settings, set the DNS servers to your Pi-hole IPs.

- **If using Omada:** Network Config --> LAN --> DHCP Server --> DNS Server: Custom
- **If using a consumer router:** Look for DHCP settings and set primary/secondary DNS

If you skipped Pi-hole, skip this step too.

### 5. Generate an Ed25519 SSH key

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
# Accept the default path (~/.ssh/id_ed25519)
# Set a passphrase (recommended)
```

### 6. Distribute the public key to all machines

```bash
ssh-copy-id user@100.x.x.x    # Use the Tailscale IP for each machine
ssh-copy-id commandervander@100.74.27.128
ssh-copy-id quartz@100.102.76.65
ssh-copy-id pi@100.72.66.9
ssh-copy-id pi@100.69.47.71
ssh-copy-id root@100.111.63.3
```

After this, you can SSH into any machine without a password.

### 7. Create the SSH config

Create `~/.ssh/config` with Match blocks for smart routing. Use the structure shown in the SSH Smart Routing section above. At minimum:

```
# Home detection
Match host server exec "~/dotfiles/scripts/is-home.sh"
    HostName ServerM2P.local

Match host server
    HostName 100.74.27.128

# Credentials
Host server
    User commandervander
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Repeat the Match/Host pattern for each machine.

### 8. Create `is-home.sh`

```bash
#!/bin/bash
# Returns 0 if on home network, 1 if remote.

# Check 1: Can we ping the gateway?
if ping -c 1 -W 1 192.168.0.1 &>/dev/null; then
    exit 0
fi

# Check 2: Are we on the home WiFi?
SSID=$(/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -I | awk '/ SSID/ {print $2}')
if [ "$SSID" = "AD6XN" ]; then
    exit 0
fi

exit 1
```

```bash
chmod +x ~/dotfiles/scripts/is-home.sh
```

**Note:** The WiFi SSID detection command above is macOS-specific. On Linux, replace it with `iwgetid -r` or `nmcli -t -f active,ssid dev wifi | grep '^yes' | cut -d: -f2`.

### 9. Test

```bash
# From home:
ssh server       # Should connect via ServerM2P.local (LAN)

# From a remote network (disconnect from home WiFi or use phone hotspot):
ssh server       # Should connect via 100.74.27.128 (Tailscale)

# Debug which hostname SSH resolved:
ssh -G server | grep hostname
```

---

## Gotchas

These are the things that will waste your time if you do not know about them upfront.

| Issue | Details | Fix |
|-------|---------|-----|
| mDNS (.local) is flaky | macOS mDNS resolution can lag or fail, especially after sleep. | Always have Tailscale IPs as the fallback in your SSH Match blocks. Never rely solely on `.local` names. |
| Pi 3B+ runs hot | The AlertNode (Pi 3B+) runs at 60-65 C and actively thermal throttles. | Use a case with a fan or add a heatsink. The Pi 5 (NetSentry) handles thermals much better. |
| `is-home.sh` must be fast | If the script takes more than 1 second, SSH will feel sluggish on every connection. | Use short ping timeouts (`-W 1`). Do not add DNS lookups or HTTP requests to the script. |
| WiFi SSID changes | If you rename your WiFi network, `is-home.sh` will think you are always remote. | Update the SSID check in `is-home.sh` to match your new network name. |
| ServerM2P uses OrbStack | ServerM2P runs containers via OrbStack, not Docker Desktop. OrbStack is lighter and has better macOS integration. | If reproducing on Linux, use standard Docker Engine instead. OrbStack is macOS-only. |
| VPS SSH prefers Tailscale | Even from home, SSH to the VPS goes through Tailscale (100.111.63.3) instead of the public IP. | This is intentional. It avoids exposing SSH on the public IP and keeps all traffic encrypted end-to-end. |
| Different users per machine | Each machine has a different username (slate, quartz, commandervander, pi, root). | The SSH config `Host` blocks define the correct `User` for each alias. You never need to remember usernames. |
