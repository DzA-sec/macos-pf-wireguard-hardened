# macOS PF Firewall · WireGuard Kill-Switch · DNSCrypt

A hardened network security stack for macOS that routes all traffic through a WireGuard VPN tunnel and encrypts DNS queries. Built on macOS-native tools, automated at boot, and designed to leave no unencrypted traffic paths.

![macOS](https://img.shields.io/badge/macOS-26.4+-black?style=flat-square)
![DNSCrypt](https://img.shields.io/badge/DNSCrypt--proxy-v2.1.15-blue?style=flat-square)
![WireGuard](https://img.shields.io/badge/WireGuard-tools-orange?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)
![Platform](https://img.shields.io/badge/platform-Apple%20Silicon%20%7C%20Intel-lightgrey?style=flat-square)

---

## 🌐 Languages
- [English](#english)
- [Deutsch](#deutsch)
- [Русский](#russian)

---

<a name="english"></a>
#  English

## Why This Project Exists

Most VPN apps protect traffic while the tunnel is active — but if the VPN disconnects, traffic silently falls back to your regular connection. DNS queries may also bypass the VPN entirely, leaking the sites you visit even when connected.

This project solves both problems:

- **Kill-switch** at the firewall level — if WireGuard goes down, all traffic is blocked, not rerouted
- **DNS isolation** — DNS queries are locked to `127.0.0.1` and encrypted via DNSCrypt-proxy, so they cannot leak through the VPN or be read in plain text
- **Anonymized DNS routing** — even the DNS resolver never sees your real IP address

It is not a replacement for a commercial VPN app. It is a self-hosted, auditable network policy that gives you direct control over what leaves your machine and how.

---

## How It Works (Simple Explanation)

When your Mac sends any network request, three things happen in order:

**1. DNS is intercepted locally**
Any app that resolves a hostname sends its DNS query to `127.0.0.1:53`. DNSCrypt-proxy receives it, encrypts it, and forwards it through a relay server to an upstream resolver. The relay sees your IP but not the query. The resolver sees the query but not your IP.

**2. The PF firewall enforces the tunnel**
The firewall blocks all outbound traffic on physical interfaces except the minimum needed to establish the WireGuard connection (DHCP, bootstrap DNS, WireGuard handshake). Everything else must go through the `utun*` tunnel interface.

**3. WireGuard carries all remaining traffic**
Once the tunnel is up, all TCP, UDP, and ICMP traffic flows through the encrypted WireGuard tunnel to your VPN server.

If WireGuard disconnects at any point, the firewall has no pass rule for the physical interface — traffic stops until the tunnel is restored.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        macOS Host                           │
│                                                             │
│  ┌──────────────┐    DNS queries     ┌──────────────────┐  │
│  │ Applications │ ─────────────────► │ DNSCrypt-proxy   │  │
│  └──────────────┘    127.0.0.1:53    │ v2.1.15          │  │
│                                      └────────┬─────────┘  │
│                                               │ Encrypted   │
│                                               ▼             │
│                                      Relay → Resolver       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                   PF Firewall                        │  │
│  │  /etc/pf.conf (system)                               │  │
│  │    └── anchor "custom"  ◄── /etc/pf.anchors/custom  │  │
│  │          ├── Default policy: block all               │  │
│  │          ├── Allow: DHCP + Bootstrap DNS + WG only   │  │
│  │          ├── KILL-SWITCH: block all uplinks          │  │
│  │          └── Pass: traffic via WireGuard (utun*)     │  │
│  └──────────────────────────┬───────────────────────────┘  │
│                             │                               │
│  ┌──────────────────────────▼───────────────────────────┐  │
│  │           WireGuard Tunnel (utun*)                   │  │
│  └──────────────────────────┬───────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                              │ Encrypted UDP tunnel
                              ▼
                     Self-hosted VPN Server
                              │
                              ▼
                          Internet
```

**Traffic paths:**
- DNS → `127.0.0.1:53` → DNSCrypt-proxy → encrypted → Relay → Resolver
- All other traffic → PF checks → WireGuard tunnel → VPN server → Internet
- WireGuard down → PF blocks everything on physical interfaces

---

## Threat Model

### What this protects against

| Threat | How |
|--------|-----|
| Traffic leaving outside VPN | PF kill-switch blocks all physical interface traffic |
| DNS queries leaking in plain text | DNS locked to loopback, encrypted via DNSCrypt |
| DNS resolver seeing your IP | Anonymized DNS routing via relay server |
| DNS hijacking by third party | Whitelist-only upstream resolvers |
| Bogon/reserved IP routing | Full RFC bogon table on all interfaces |
| TCP port scanning | Invalid flag combinations (NULL, FIN, XMAS, SYN-FIN) blocked |
| QUIC/DoQ bypassing DNS controls | UDP 443/853 blocked inside VPN tunnel |
| IPv6 bypassing the tunnel | IPv6 blocked globally on non-loopback interfaces |
| State table exhaustion | Explicit limits and aggressive timeout optimization |
| IP addresses in local logs | Log-level IP encryption via `ipcrypt-ndx` |

### What this does NOT protect against

- Compromised VPN server — the server operator can still see your traffic destinations
- Malware with root access on your Mac — it can modify PF rules
- Browser fingerprinting, cookies, or account-based tracking
- Attacks on WireGuard itself (protocol-level vulnerabilities)
- Physical access to your machine

This configuration controls **network policy**. It does not replace endpoint security, browser privacy measures, or VPN server trust decisions.

---

## Limitations and Trade-offs

| Feature | Status | Reason |
|---------|--------|--------|
| HTTP/3 (QUIC) | Disabled | UDP/443 blocked — browsers fall back to HTTP/2 automatically |
| IPv6 connectivity | Disabled | Tunnel handles IPv4 only — enables IPv6 would risk bypass |
| AirDrop / mDNS / Bonjour | Disabled | Prevents local network presence broadcasting |
| DNS-over-Tor | Optional, off by default | Adds ~200–500ms latency; use only if relay trust is a concern |

---

## Features

**Security**
- PF kill-switch with default-deny policy on all physical interfaces
- DNS isolation — all queries routed through `127.0.0.1` exclusively
- Anonymized DNS — resolver sees relay IP only, relay sees client IP only
- Bogon filtering on Wi-Fi and VPN interfaces
- TCP hardening against NULL, FIN, XMAS, and SYN-FIN scans
- IPv6 disabled globally to prevent tunnel bypass

**Operations**
- Boot persistence via LaunchDaemon with `WatchPaths` — auto-reloads on `pf.conf` change
- Rate limiting with dynamic `<abusers>` table
- Structured logging: separate files for PF boot, blocked traffic, DNS queries
- Leak verification script — DNS, IPv6, WebRTC, kill-switch

---

## Technologies Used

| Technology | Role | Version |
|-----------|------|---------|
| **PF (Packet Filter)** | Stateful firewall, kill-switch, traffic policy | macOS built-in |
| **WireGuard** | VPN tunnel protocol | wireguard-tools + wireguard-go |
| **DNSCrypt-proxy** | Encrypted DNS, Anonymized DNS routing | v2.1.15 |
| **launchd / LaunchDaemon** | Service management, boot persistence | macOS built-in |
| **Homebrew** | Package management | 4.x |

---

---

## Repository Structure

```
macos-pf-wireguard-hardened/
├── pf/
│   └── pf.conf                        # Custom PF anchor — kill-switch + DNS isolation
├── dnscrypt/
│   ├── dnscrypt-proxy.toml            # DNSCrypt-proxy configuration
│   ├── blocked-names.txt              # Domain blocklist
│   └── blocked-ips.txt                # IP blocklist
├── launchdaemons/
│   └── com.user.pfctl.enable.plist    # LaunchDaemon — boot persistence + WatchPaths
├── scripts/
│   └── check-leaks.sh                 # Automated leak verification
└── README.md
```

---

## Quick Start

Minimum steps to get running. For full detail, see [Full Installation Guide](#full-installation-guide).

```bash
# 1. Install dependencies
brew install wireguard-tools wireguard-go dnscrypt-proxy

# 2. Generate WireGuard keys
wg genkey | tee privatekey | wg pubkey > publickey
# Copy privatekey → PrivateKey in wg0.conf
# Send publickey → your server admin

# 3. Create WireGuard config (Apple Silicon path)
sudo mkdir -p /opt/homebrew/etc/wireguard
sudo nano /opt/homebrew/etc/wireguard/wg0.conf
# Fill in: PrivateKey, Address, DNS=127.0.0.1, PublicKey, Endpoint, AllowedIPs=0.0.0.0/0
sudo chmod 600 /opt/homebrew/etc/wireguard/wg0.conf

# 4. Configure DNSCrypt
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
# Edit: set routes in [anonymized_dns], generate key with: openssl rand -hex 32
brew services start dnscrypt-proxy

# 5. Set system DNS to 127.0.0.1
# System Settings → Network → Wi-Fi → Details → DNS → 127.0.0.1

# 6. Configure and load PF
# Edit pf/pf.conf — fill in wg_endpoints, bootstrap_dns, dnscrypt_upstream, wg_internal
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf   # validate
sudo pfctl -f /etc/pf.conf && sudo pfctl -e

# 7. Install LaunchDaemon (boot persistence)
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist

# 8. Connect and verify
sudo wg-quick up wg0
bash scripts/check-leaks.sh
```

---

## Full Installation Guide

### Prerequisites

- macOS 26.4 or later
- A running WireGuard server with: public key · server IP and port · assigned VPN subnet
- Terminal (Applications → Utilities → Terminal)

---

### Step 1 — Install Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew --version
```

---

### Step 2 — Install and Configure WireGuard

```bash
brew install wireguard-tools wireguard-go
```

**Generate key pair:**

```bash
# Apple Silicon
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey | wg pubkey > /opt/homebrew/etc/wireguard/publickey

cat /opt/homebrew/etc/wireguard/privatekey   # → paste into PrivateKey
cat /opt/homebrew/etc/wireguard/publickey    # → send to server admin

rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey
```

> Config paths: Apple Silicon → `/opt/homebrew/etc/wireguard/` · Intel → `/usr/local/etc/wireguard/`
> Check chip: `which wg-quick` — path prefix indicates chip type

**Create tunnel config:**

```bash
sudo nano /opt/homebrew/etc/wireguard/wg0.conf
```

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address    = 10.8.0.2/24
DNS        = 127.0.0.1          # Points to DNSCrypt-proxy — do not change

[Peer]
PublicKey           = SERVER_PUBLIC_KEY
Endpoint            = SERVER_IP:SERVER_PORT
AllowedIPs          = 0.0.0.0/0
PersistentKeepalive = 25
```

```bash
sudo chmod 600 /opt/homebrew/etc/wireguard/wg0.conf
sudo wg-quick up wg0 && sudo wg show
```

---

### Step 3 — Install DNSCrypt-proxy

```bash
brew install dnscrypt-proxy
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
```

**Configure servers and relays** — browse https://dnscrypt.info/public-servers

Select servers with: `DNSCrypt` protocol · `No logs` · `DNSSEC`

Select relays operated by a **different organization** and in a **different country** than the server.

```bash
nano /opt/homebrew/etc/dnscrypt-proxy.toml
```

```toml
# [anonymized_dns] section
routes = [
    { server_name='YOUR_SERVER', via=['YOUR_RELAY_1', 'YOUR_RELAY_2'] },
]
```

**Generate log encryption key:**

```bash
openssl rand -hex 32   # paste into key = "" under [ip_encryption]
```

**Copy blocklists and start:**

```bash
mkdir -p /opt/homebrew/etc/dnscrypt-proxy
cp dnscrypt/blocked-names.txt /opt/homebrew/etc/dnscrypt-proxy/
cp dnscrypt/blocked-ips.txt   /opt/homebrew/etc/dnscrypt-proxy/

brew services start dnscrypt-proxy
dig google.com @127.0.0.1
```

**Set system DNS:** System Settings → Network → Wi-Fi → Details → DNS → set `127.0.0.1`, remove all others.

---

### Step 4 — Configure PF Firewall

**What the PF rules do (high-level):**
- Default policy: block everything in and out
- Before tunnel: allow only DHCP, whitelisted bootstrap DNS, and WireGuard handshake packets
- Kill-switch: block all traffic on physical uplink interfaces after those exceptions
- Inside tunnel: allow all traffic through `utun*`, with rate limiting and bogon filtering

**Fill in `pf/pf.conf` tables:**

```sh
table <wg_endpoints>      persist { SERVER_IP }           # [Peer] Endpoint IP
table <bootstrap_dns>     persist { 1.1.1.1 }             # Pre-tunnel DNS (e.g. Cloudflare)
table <dnscrypt_upstream> persist { DNSCRYPT_SERVER_IP }  # resolve: dig +short HOSTNAME
table <wg_internal>       persist { 10.8.0.0/24 }         # [Interface] Address with last octet → 0
```

Update WireGuard port:

```sh
pass out log quick on $wifi proto udp to <wg_endpoints> port { SERVER_PORT } keep state
```

**Add custom anchor to system `/etc/pf.conf`:**

```bash
grep -n "custom" /etc/pf.conf   # check if already present
sudo cp /etc/pf.conf /etc/pf.conf.backup
sudo nano /etc/pf.conf
```

Insert **before** `anchor "com.apple/*"`:

```sh
# Custom rules run first — kill-switch and DNS policy take priority
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"
```

**Install and load:**

```bash
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf   # validate — no output means no errors
sudo pfctl -f /etc/pf.conf && sudo pfctl -e
sudo pfctl -a custom -sr | head -20
```

---

### Step 5 — Install LaunchDaemon

Installs PF rules automatically at every boot with a 5-second delay to allow network interfaces to initialize. WatchPaths triggers a reload if `pf.conf` is modified.

```bash
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist

sudo launchctl list | grep pfctl
cat /var/log/pf_autoload.log
```

---

### Step 6 — Verify

```bash
sudo wg-quick up wg0
bash scripts/check-leaks.sh
```

Online verification:
- https://dnsleaktest.com — Extended test
- https://ipleak.net
- https://browserleaks.com/webrtc

**Kill-switch test:**

```bash
curl https://ifconfig.me               # confirm VPN server IP shown
sudo wg-quick down wg0                 # disconnect
curl --max-time 5 https://ifconfig.me  # must timeout — no fallback traffic
sudo wg-quick up wg0                   # reconnect
```

---

### Optional: DNS over Tor

PF already includes loopback rules for Tor ports 9050/9150/9151 — no PF changes needed.

```bash
brew install tor && brew services start tor
# In dnscrypt-proxy.toml, uncomment:
# proxy = 'socks5://dnscrypt:dnscrypt@127.0.0.1:9050'
brew services restart dnscrypt-proxy
dig google.com @127.0.0.1
```

Adds ~200–500ms per DNS query. Recommended only if you also want to hide DNS traffic from relay operators.

---

### Troubleshooting

| Symptom | Command |
|---------|---------|
| No internet after PF load | `sudo pfctl -n -f /etc/pf.conf` — check for syntax errors |
| Emergency — restore access | `sudo pfctl -d` — disables PF completely |
| DNS not resolving | `brew services restart dnscrypt-proxy` + `dig google.com @127.0.0.1` |
| WireGuard not connecting | `sudo wg show` + verify Endpoint IP/port in pf.conf tables |
| PF rules lost after reboot | `cat /var/log/pf_autoload_error.log` |
| Verify DNS resolver in use | `scutil --dns \| grep nameserver` — must show `127.0.0.1` |

---

### Useful Commands

```bash
# WireGuard
sudo wg-quick up wg0 / down wg0        # connect / disconnect
sudo wg show                            # tunnel status and peer stats

# PF
sudo pfctl -e / -d                      # enable / disable
sudo pfctl -f /etc/pf.conf             # reload rules
sudo pfctl -n -f /etc/pf.conf         # validate syntax without loading
sudo pfctl -a custom -sr               # show loaded custom anchor rules
sudo pfctl -ss | grep utun             # active states on VPN interface
sudo pfctl -t abusers -T show          # IPs currently rate-limited
sudo pfctl -t abusers -T flush         # clear abusers table

# DNSCrypt
brew services restart dnscrypt-proxy
dig google.com @127.0.0.1
dig +short TXT whoami.ds.akahelp.net   # check which resolver is answering

# Logs
tail -f /var/log/pf_autoload.log
tail -f /opt/homebrew/etc/dnscrypt-proxy.log
sudo tcpdump -n -i en0 port 53         # should be empty — DNS stays on loopback
```

---

## Contributing

Issues and pull requests are welcome. Please open an issue before submitting a PR to discuss the proposed change.

---

<a name="deutsch"></a>
#  Deutsch

## Warum dieses Projekt existiert

Die meisten VPN-Apps schützen den Traffic nur solange der Tunnel aktiv ist. Trennt sich die Verbindung, fließt der Traffic still und leise über die normale Verbindung weiter. DNS-Anfragen können den VPN-Tunnel vollständig umgehen und die besuchten Seiten verraten — selbst bei aktiver Verbindung.

Dieses Projekt löst beide Probleme:

- **Kill-Switch** auf Firewall-Ebene — fällt WireGuard aus, wird der gesamte Traffic blockiert, nicht umgeleitet
- **DNS-Isolation** — DNS-Anfragen sind auf `127.0.0.1` beschränkt und via DNSCrypt-proxy verschlüsselt
- **Anonymized DNS** — selbst der DNS-Resolver sieht nie die echte IP-Adresse

Es ist kein Ersatz für eine kommerzielle VPN-App. Es ist eine selbst gehostete, prüfbare Netzwerkrichtlinie, die volle Kontrolle darüber gibt, was das Gerät verlässt und wie.

---

## Wie es funktioniert (vereinfacht)

**1. DNS wird lokal abgefangen**
Jede Namensauflösung geht an `127.0.0.1:53`. DNSCrypt-proxy empfängt, verschlüsselt und leitet sie über einen Relay-Server weiter. Der Relay sieht die IP, nicht die Anfrage. Der Resolver sieht die Anfrage, nicht die IP.

**2. PF erzwingt den Tunnel**
Die Firewall blockiert den gesamten ausgehenden Traffic auf physischen Interfaces — außer dem Minimum für den WireGuard-Verbindungsaufbau (DHCP, Bootstrap-DNS, WG-Handshake).

**3. WireGuard trägt den restlichen Traffic**
Sobald der Tunnel aktiv ist, fließt alles — TCP, UDP, ICMP — verschlüsselt zum VPN-Server.

Trennt sich WireGuard, gibt es keine Pass-Regel für das physische Interface — Traffic stoppt, bis der Tunnel wiederhergestellt ist.

---

## Bedrohungsmodell

### Wovor es schützt

| Bedrohung | Schutz |
|-----------|--------|
| Traffic außerhalb VPN | PF Kill-Switch blockiert alle physischen Interfaces |
| DNS-Leak im Klartext | DNS auf Loopback gesperrt, via DNSCrypt verschlüsselt |
| Resolver sieht Client-IP | Anonymized DNS über Relay-Server |
| DNS-Hijacking | Nur Whitelist-Upstreams erlaubt |
| Bogon-IP-Routing | Vollständige RFC-Bogon-Tabelle |
| TCP-Port-Scanning | Ungültige Flag-Kombinationen blockiert |
| IPv6-Bypass | IPv6 global auf Non-Loopback-Interfaces deaktiviert |

### Wovor es NICHT schützt

- Kompromittierter VPN-Server
- Malware mit Root-Zugriff auf dem Mac
- Browser-Fingerprinting oder Cookie-basiertes Tracking
- Angriffe auf WireGuard selbst

---

## Einschränkungen

| Feature | Status | Grund |
|---------|--------|-------|
| HTTP/3 (QUIC) | Deaktiviert | UDP/443 blockiert — Browser wechseln automatisch auf HTTP/2 |
| IPv6 | Deaktiviert | Tunnel nur IPv4 — IPv6 würde Bypass ermöglichen |
| AirDrop / mDNS / Bonjour | Deaktiviert | Verhindert lokale Netzwerk-Broadcasts |
| DNS-over-Tor | Optional, standardmäßig aus | ~200–500ms Latenz; nur bei Relay-Misstrauen sinnvoll |

---

## Verwendete Technologien

| Technologie | Rolle |
|-------------|-------|
| **PF (Packet Filter)** | Stateful Firewall, Kill-Switch, Traffic-Policy |
| **WireGuard** | VPN-Tunnelprotokoll |
| **DNSCrypt-proxy v2.1.15** | Verschlüsseltes DNS, Anonymized-DNS-Routing |
| **launchd / LaunchDaemon** | Service-Management, Boot-Persistenz |
| **Homebrew** | Paketverwaltung |

---

---

## Quick Start

```bash
# 1. Abhängigkeiten installieren
brew install wireguard-tools wireguard-go dnscrypt-proxy

# 2. WireGuard-Schlüssel generieren
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey | wg pubkey > /opt/homebrew/etc/wireguard/publickey
# privatekey → PrivateKey in wg0.conf
# publickey → an Server-Admin senden
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey

# 3. WireGuard konfigurieren
sudo nano /opt/homebrew/etc/wireguard/wg0.conf
# PrivateKey, Address, DNS=127.0.0.1, PublicKey, Endpoint, AllowedIPs=0.0.0.0/0
sudo chmod 600 /opt/homebrew/etc/wireguard/wg0.conf

# 4. DNSCrypt konfigurieren
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
# routes in [anonymized_dns] setzen, Schlüssel generieren: openssl rand -hex 32
brew services start dnscrypt-proxy

# 5. System-DNS auf 127.0.0.1 setzen
# Systemeinstellungen → Netzwerk → WLAN → Details → DNS → 127.0.0.1

# 6. PF konfigurieren und laden
# pf/pf.conf Tabellen ausfüllen: wg_endpoints, bootstrap_dns, dnscrypt_upstream, wg_internal
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf && sudo pfctl -f /etc/pf.conf && sudo pfctl -e

# 7. LaunchDaemon installieren
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist

# 8. Verbinden und prüfen
sudo wg-quick up wg0
bash scripts/check-leaks.sh
```

---

## Vollständige Installationsanleitung

### Voraussetzungen

- macOS 26.4 oder neuer
- Laufender WireGuard-Server mit: öffentlichem Schlüssel · IP + Port · VPN-Subnetz
- Terminal (Programme → Dienstprogramme → Terminal)

### Schritt 1 — Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew --version
```

### Schritt 2 — WireGuard

```bash
brew install wireguard-tools wireguard-go
```

Schlüsselpaar generieren:

```bash
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey | wg pubkey > /opt/homebrew/etc/wireguard/publickey
cat /opt/homebrew/etc/wireguard/privatekey   # → PrivateKey
cat /opt/homebrew/etc/wireguard/publickey    # → an Server senden
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey
```

> Pfad: Apple Silicon → `/opt/homebrew/etc/wireguard/` · Intel → `/usr/local/etc/wireguard/`
> Chip prüfen: `which wg-quick`

Tunnel-Konfiguration:

```bash
sudo nano /opt/homebrew/etc/wireguard/wg0.conf
```

```ini
[Interface]
PrivateKey = IHR_PRIVATER_SCHLÜSSEL
Address    = 10.8.0.2/24
DNS        = 127.0.0.1

[Peer]
PublicKey           = ÖFFENTLICHER_SERVER_SCHLÜSSEL
Endpoint            = SERVER_IP:SERVER_PORT
AllowedIPs          = 0.0.0.0/0
PersistentKeepalive = 25
```

```bash
sudo chmod 600 /opt/homebrew/etc/wireguard/wg0.conf
sudo wg-quick up wg0 && sudo wg show
```

### Schritt 3 — DNSCrypt-proxy

```bash
brew install dnscrypt-proxy
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
nano /opt/homebrew/etc/dnscrypt-proxy.toml
```

Server: https://dnscrypt.info/public-servers — Protokoll `DNSCrypt`, `No logs` ✓, `DNSSEC` ✓

Relays: andere Organisation + anderes Land als der Server.

```toml
routes = [
    { server_name='IHR_SERVER', via=['IHR_RELAY_1', 'IHR_RELAY_2'] },
]
```

```bash
openssl rand -hex 32   # → key = "" unter [ip_encryption]
mkdir -p /opt/homebrew/etc/dnscrypt-proxy
cp dnscrypt/blocked-*.txt /opt/homebrew/etc/dnscrypt-proxy/
brew services start dnscrypt-proxy
dig google.com @127.0.0.1
```

System-DNS: **Systemeinstellungen → Netzwerk → WLAN → Details → DNS** → `127.0.0.1`, alle anderen entfernen.

### Schritt 4 — PF-Firewall

**Was die PF-Regeln tun (vereinfacht):**
- Standard: alles blockieren
- Vor dem Tunnel: nur DHCP, Bootstrap-DNS und WireGuard-Handshake erlaubt
- Kill-Switch: alle physischen Interfaces nach diesen Ausnahmen blockiert
- Im Tunnel: alles über `utun*`, mit Rate-Limiting und Bogon-Filterung

`pf/pf.conf` Tabellen ausfüllen:

```sh
table <wg_endpoints>      persist { SERVER_IP }
table <bootstrap_dns>     persist { 1.1.1.1 }
table <dnscrypt_upstream> persist { DNSCRYPT_SERVER_IP }
table <wg_internal>       persist { 10.8.0.0/24 }
```

Port anpassen:

```sh
pass out log quick on $wifi proto udp to <wg_endpoints> port { SERVER_PORT } keep state
```

Custom Anchor in `/etc/pf.conf` (vor `anchor "com.apple/*"`):

```bash
sudo cp /etc/pf.conf /etc/pf.conf.backup
sudo nano /etc/pf.conf
```

```sh
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"
```

```bash
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf
sudo pfctl -f /etc/pf.conf && sudo pfctl -e
```

### Schritt 5 — LaunchDaemon

```bash
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist
```

### Schritt 6 — Verifizierung

```bash
sudo wg-quick up wg0
bash scripts/check-leaks.sh
```

Kill-Switch testen:

```bash
curl https://ifconfig.me
sudo wg-quick down wg0
curl --max-time 5 https://ifconfig.me   # muss Timeout ergeben
sudo wg-quick up wg0
```

### Fehlerbehebung

| Problem | Befehl |
|---------|--------|
| Kein Internet nach PF-Laden | `sudo pfctl -n -f /etc/pf.conf` |
| Notfall — PF deaktivieren | `sudo pfctl -d` |
| DNS wird nicht aufgelöst | `brew services restart dnscrypt-proxy` |
| WireGuard verbindet nicht | `sudo wg show` |
| PF-Regeln nach Neustart verloren | `cat /var/log/pf_autoload_error.log` |

---

<a name="russian"></a>
# Русский

## Почему этот проект существует

Большинство VPN приложений защищают трафик пока туннель активен — но если VPN отключается, трафик молча переходит на обычное соединение. DNS запросы могут полностью обходить VPN туннель, раскрывая посещаемые сайты даже при активном подключении.

Этот проект решает обе проблемы:

- **Kill-switch** на уровне файрвола — если WireGuard падает, весь трафик блокируется, а не перенаправляется
- **DNS изоляция** — DNS запросы ограничены `127.0.0.1` и зашифрованы через DNSCrypt-proxy
- **Anonymized DNS** — даже DNS резолвер никогда не видит реальный IP адрес

Это не замена коммерческому VPN приложению. Это самостоятельно размещённая, поддающаяся аудиту сетевая политика, дающая прямой контроль над тем, что покидает машину и каким образом.

---

## Как это работает (простое объяснение)

**1. DNS перехватывается локально**
Любое разрешение имён идёт на `127.0.0.1:53`. DNSCrypt-proxy получает запрос, шифрует его и пересылает через relay сервер к upstream резолверу. Relay видит IP, но не запрос. Резолвер видит запрос, но не IP.

**2. PF принудительно направляет через туннель**
Файрвол блокирует весь исходящий трафик на физических интерфейсах — кроме минимума необходимого для установки WireGuard соединения (DHCP, bootstrap DNS, WG handshake).

**3. WireGuard несёт весь остальной трафик**
Как только туннель поднят, весь TCP, UDP и ICMP трафик проходит через зашифрованный WireGuard туннель к VPN серверу.

Если WireGuard отключается — нет pass правила для физического интерфейса — трафик останавливается до восстановления туннеля.

---

## Модель угроз

### От чего защищает

| Угроза | Защита |
|--------|--------|
| Трафик вне VPN | PF kill-switch блокирует все физические интерфейсы |
| DNS запросы в открытом виде | DNS на loopback, зашифрован через DNSCrypt |
| Резолвер видит IP клиента | Anonymized DNS через relay сервер |
| DNS hijacking | Только whitelist upstream конфигурация |
| Bogon IP маршрутизация | Полная RFC bogon таблица |
| TCP сканирование портов | Невалидные комбинации флагов заблокированы |
| IPv6 обход туннеля | IPv6 заблокирован глобально на non-loopback интерфейсах |

### От чего НЕ защищает

- Скомпрометированный VPN сервер
- Вредоносное ПО с root доступом на Mac
- Browser fingerprinting или cookie-based отслеживание
- Атаки на сам протокол WireGuard

---

## Ограничения и компромиссы

| Функция | Статус | Причина |
|---------|--------|---------|
| HTTP/3 (QUIC) | Отключён | UDP/443 заблокирован — браузеры автоматически переходят на HTTP/2 |
| IPv6 | Отключён | Туннель только IPv4 — IPv6 открыл бы путь для обхода |
| AirDrop / mDNS / Bonjour | Отключён | Предотвращает широковещательные объявления в локальной сети |
| DNS через Tor | Опционально, по умолчанию выключен | ~200–500мс задержки; только если relay операторы вызывают беспокойство |

---

## Используемые технологии

| Технология | Роль |
|-----------|------|
| **PF (Packet Filter)** | Stateful файрвол, kill-switch, политика трафика |
| **WireGuard** | Протокол VPN туннеля |
| **DNSCrypt-proxy v2.1.15** | Зашифрованный DNS, Anonymized DNS маршрутизация |
| **launchd / LaunchDaemon** | Управление службами, сохранение при загрузке |
| **Homebrew** | Управление пакетами |

---

---

## Quick Start

```bash
# 1. Установить зависимости
brew install wireguard-tools wireguard-go dnscrypt-proxy

# 2. Сгенерировать ключи WireGuard
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey | wg pubkey > /opt/homebrew/etc/wireguard/publickey
# privatekey → вставить в PrivateKey в wg0.conf
# publickey → отправить администратору сервера
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey

# 3. Создать конфиг WireGuard
sudo nano /opt/homebrew/etc/wireguard/wg0.conf
# Заполнить: PrivateKey, Address, DNS=127.0.0.1, PublicKey, Endpoint, AllowedIPs=0.0.0.0/0
sudo chmod 600 /opt/homebrew/etc/wireguard/wg0.conf

# 4. Настроить DNSCrypt
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
# Заполнить routes в [anonymized_dns], сгенерировать ключ: openssl rand -hex 32
brew services start dnscrypt-proxy

# 5. Установить системный DNS на 127.0.0.1
# Системные настройки → Сеть → Wi-Fi → Подробнее → DNS → 127.0.0.1

# 6. Настроить и загрузить PF
# Заполнить таблицы в pf/pf.conf: wg_endpoints, bootstrap_dns, dnscrypt_upstream, wg_internal
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf && sudo pfctl -f /etc/pf.conf && sudo pfctl -e

# 7. Установить LaunchDaemon
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist

# 8. Подключить и проверить
sudo wg-quick up wg0
bash scripts/check-leaks.sh
```

---

## Полная инструкция по установке

### Предварительные требования

- macOS 26.4 или новее
- Работающий WireGuard сервер с: публичным ключом · IP и портом · назначенной VPN подсетью
- Терминал (Программы → Утилиты → Терминал)

### Шаг 1 — Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew --version
```

### Шаг 2 — WireGuard

```bash
brew install wireguard-tools wireguard-go
```

Генерация ключей:

```bash
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey | wg pubkey > /opt/homebrew/etc/wireguard/publickey
cat /opt/homebrew/etc/wireguard/privatekey   # → вставить в PrivateKey
cat /opt/homebrew/etc/wireguard/publickey    # → отправить на сервер
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey
```

> Путь: Apple Silicon → `/opt/homebrew/etc/wireguard/` · Intel → `/usr/local/etc/wireguard/`
> Определить чип: `which wg-quick`

Конфигурация туннеля:

```bash
sudo nano /opt/homebrew/etc/wireguard/wg0.conf
```

```ini
[Interface]
PrivateKey = ВАШ_ПРИВАТНЫЙ_КЛЮЧ
Address    = 10.8.0.2/24
DNS        = 127.0.0.1          # Указывает на DNSCrypt-proxy — не менять

[Peer]
PublicKey           = ПУБЛИЧНЫЙ_КЛЮЧ_СЕРВЕРА
Endpoint            = IP_СЕРВЕРА:ПОРТ_СЕРВЕРА
AllowedIPs          = 0.0.0.0/0
PersistentKeepalive = 25
```

```bash
sudo chmod 600 /opt/homebrew/etc/wireguard/wg0.conf
sudo wg-quick up wg0 && sudo wg show
```

### Шаг 3 — DNSCrypt-proxy

```bash
brew install dnscrypt-proxy
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
nano /opt/homebrew/etc/dnscrypt-proxy.toml
```

Серверы: https://dnscrypt.info/public-servers — протокол `DNSCrypt`, `No logs` ✓, `DNSSEC` ✓

Relay: другая организация + другая страна чем у сервера.

```toml
routes = [
    { server_name='ВАШ_СЕРВЕР', via=['ВАШ_RELAY_1', 'ВАШ_RELAY_2'] },
]
```

```bash
openssl rand -hex 32   # → вставить в key = "" под [ip_encryption]
mkdir -p /opt/homebrew/etc/dnscrypt-proxy
cp dnscrypt/blocked-*.txt /opt/homebrew/etc/dnscrypt-proxy/
brew services start dnscrypt-proxy
dig google.com @127.0.0.1
```

Системный DNS: **Системные настройки → Сеть → Wi-Fi → Подробнее → DNS** → `127.0.0.1`, удалить остальные.

### Шаг 4 — PF Firewall

**Что делают правила PF (кратко):**
- По умолчанию: блокировать всё
- До туннеля: разрешить только DHCP, whitelisted bootstrap DNS и WireGuard handshake
- Kill-switch: заблокировать все физические интерфейсы после этих исключений
- В туннеле: разрешить всё через `utun*`, с rate limiting и bogon фильтрацией

Заполнить таблицы в `pf/pf.conf`:

```sh
table <wg_endpoints>      persist { IP_СЕРВЕРА }
table <bootstrap_dns>     persist { 1.1.1.1 }
table <dnscrypt_upstream> persist { IP_DNSCRYPT_СЕРВЕРА }
table <wg_internal>       persist { 10.8.0.0/24 }
```

Обновить порт:

```sh
pass out log quick on $wifi proto udp to <wg_endpoints> port { ПОРТ_СЕРВЕРА } keep state
```

Добавить custom anchor в `/etc/pf.conf` (перед `anchor "com.apple/*"`):

```bash
sudo cp /etc/pf.conf /etc/pf.conf.backup
sudo nano /etc/pf.conf
```

```sh
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"
```

```bash
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf
sudo pfctl -f /etc/pf.conf && sudo pfctl -e
```

### Шаг 5 — LaunchDaemon

Автоматически устанавливает правила PF при каждой загрузке. WatchPaths перезагружает правила при изменении `pf.conf`.

```bash
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist
```

### Шаг 6 — Верификация

```bash
sudo wg-quick up wg0
bash scripts/check-leaks.sh
```

Тест kill-switch:

```bash
curl https://ifconfig.me                    # должен показать IP VPN сервера
sudo wg-quick down wg0                      # отключить
curl --max-time 5 https://ifconfig.me       # должен вернуть таймаут
sudo wg-quick up wg0
```

### Устранение неполадок

| Проблема | Команда |
|---------|---------|
| Нет интернета после загрузки PF | `sudo pfctl -n -f /etc/pf.conf` |
| Аварийное восстановление | `sudo pfctl -d` |
| DNS не резолвится | `brew services restart dnscrypt-proxy` |
| WireGuard не подключается | `sudo wg show` |
| Правила PF сброшены после перезагрузки | `cat /var/log/pf_autoload_error.log` |

---

## License

MIT — free to use, modify, and distribute.

## Contributing

Issues and pull requests are welcome. Please open an issue before submitting a PR to discuss the proposed change.

---

*Tested on: macOS 26.4 · DNSCrypt-proxy v2.1.15 · wireguard-tools · wireguard-go*
