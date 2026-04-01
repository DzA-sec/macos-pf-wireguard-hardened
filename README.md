# macOS PF Firewall · WireGuard Kill-Switch · DNSCrypt

> Hardened macOS network security stack: PF firewall with strict WireGuard VPN kill-switch, encrypted DNS via DNSCrypt-proxy, and complete traffic leak prevention.

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
# 🇬🇧 English

## Goal

This configuration is designed for **privacy-conscious users, security researchers, and IT professionals** who prefer full control over their network stack.

The goal is to ensure that **all network traffic always goes through a WireGuard VPN tunnel** — with zero possibility of leaking outside it. All DNS queries are encrypted via DNSCrypt-proxy v2.1.15, so DNS traffic cannot be intercepted or monitored in plain text.

## How It Works

```
┌─────────────────────────────────────────────────────┐
│                     macOS Host                      │
│                                                     │
│  Applications (browser, mail, etc.)                 │
│       │                                             │
│       │ All DNS queries → 127.0.0.1:53              │
│       ▼                                             │
│  DNSCrypt-proxy v2.1.15                             │
│  Encrypts DNS queries → sends via Relay → Resolver  │
│       │                                             │
│       ▼                                             │
│  PF Firewall — custom anchor                        │
│  ├── Default policy: block ALL traffic              │
│  ├── Allow: DHCP, Bootstrap DNS, WG handshake only  │
│  ├── KILL-SWITCH: block everything on all NICs      │
│  └── Allow: traffic ONLY through WireGuard (utun*)  │
│       │                                             │
│       ▼                                             │
│  WireGuard Tunnel (utun*)                           │
└──────────────────────┬──────────────────────────────┘
                       │ Encrypted tunnel
                       ▼
              VPN Server (self-hosted)
                       │
                       ▼
                   Internet
```

## What This Protects Against

| Threat | Protection |
|--------|-----------|
| Traffic leak outside VPN | Kill-switch blocks all non-tunnel traffic |
| DNS leak | DNS blocked on all interfaces except 127.0.0.1 |
| DNS interception | All DNS queries encrypted via DNSCrypt |
| DNS hijacking | DNSCrypt with whitelisted upstreams only |
| Resolver seeing your real IP | Anonymized DNS routes through relay servers |
| Bogon/reserved IP traffic | Bogon table blocks all RFC-reserved ranges |
| TCP port scanning | Invalid TCP flag combinations blocked |
| QUIC/DoQ DNS bypass | UDP 443/853 blocked inside VPN |
| IPv6 leak | IPv6 blocked on all non-loopback interfaces |
| mDNS/SSDP/NetBIOS leakage | Blocked on Wi-Fi interface |
| IP addresses in log files | Client IP encryption via ipcrypt-ndx |

## Intentionally Disabled

| Feature | Reason |
|---------|--------|
| HTTP/3 (QUIC) | UDP 443 blocked — deliberate trade-off, sites fall back to HTTP/2 automatically |
| IPv6 | Blocked globally — prevents IPv6 tunnel bypass |
| mDNS / AirDrop / Bonjour | Blocked — prevents local network presence broadcasting |

## Repository Structure

```
macos-pf-wireguard-hardened/
├── README.md
├── pf/
│   └── pf.conf                        # Custom PF anchor rules
├── dnscrypt/
│   ├── dnscrypt-proxy.toml            # DNSCrypt-proxy v2.1.15 configuration
│   ├── blocked-names.txt              # Domain blocklist (customize)
│   └── blocked-ips.txt                # IP blocklist (customize)
├── launchdaemons/
│   └── com.user.pfctl.enable.plist    # PF autostart at boot
└── scripts/
    └── check-leaks.sh                 # DNS/IP leak verification script
```

---

## Installation Overview

```
Step 1: Install Homebrew
    ↓
Step 2: Install WireGuard → generate keys → create config
    ↓
Step 3: Install DNSCrypt-proxy
    ↓
Step 4: Configure DNSCrypt-proxy → choose servers → generate key
    ↓
Step 5: Configure PF firewall → fill in IP tables → add custom anchor
    ↓
Step 6: Install LaunchDaemon (PF autostart at boot)
    ↓
Step 7: Connect WireGuard → verify no leaks
```

## Requirements

Before you start, make sure you have:

- macOS 26.4 or later
- A running WireGuard **server** with:
  - ✓ Server public key
  - ✓ Server IP address and port
  - ✓ Internal VPN subnet it assigns to clients (e.g. `10.8.0.0/24`)
- Terminal (Applications → Utilities → Terminal)

---

## Full Installation Guide

### Step 1 — Install Homebrew

Homebrew is the package manager for macOS. Open **Terminal** and run:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

The installer will ask for your macOS password — type it and press Enter (no characters will appear, that is normal).

After installation, follow any on-screen instructions (usually adding Homebrew to your PATH). Then verify:

```bash
brew --version
# Expected output: Homebrew 4.x.x
```

---

### Step 2 — Install WireGuard

```bash
brew install wireguard-tools wireguard-go
```

Verify installation:

```bash
wg --version
# Expected output: wireguard-tools vX.X.X
```

#### 2.1 Generate WireGuard key pair

Before creating your config file, generate a private/public key pair:

```bash
# Generate keys and save to temporary files
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey | wg pubkey > /opt/homebrew/etc/wireguard/publickey

# View your private key — copy this, you will need it for PrivateKey =
cat /opt/homebrew/etc/wireguard/privatekey

# View your public key — send this to your server administrator
# The server needs it to create a [Peer] entry for your client
cat /opt/homebrew/etc/wireguard/publickey

# Delete the key files after copying — they are no longer needed
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey
```

> **Important:** Never share your private key with anyone. It cannot be recovered if lost — generate a new pair instead.

#### 2.2 Create your WireGuard config file

When installed via Homebrew, WireGuard config files are stored at:

| Mac chip | Config path |
|----------|-------------|
| **Apple Silicon (M1/M2/M3/M4)** | `/opt/homebrew/etc/wireguard/` |
| Intel | `/usr/local/etc/wireguard/` |

> **Note:** This is different from Linux where configs live in `/etc/wireguard/`. Check your chip:
> ```bash
> which wg-quick
> # /opt/homebrew/bin/wg-quick → Apple Silicon
> # /usr/local/bin/wg-quick   → Intel
> ```

Create the config file:

```bash
# Apple Silicon (M1/M2/M3/M4)
sudo mkdir -p /opt/homebrew/etc/wireguard
sudo nano /opt/homebrew/etc/wireguard/wg0.conf

# Intel Mac — use this instead:
# sudo mkdir -p /usr/local/etc/wireguard
# sudo nano /usr/local/etc/wireguard/wg0.conf
```

Config file structure:

```ini
[Interface]
# Your private key generated in step 2.1
PrivateKey = YOUR_PRIVATE_KEY_HERE

# Your internal VPN IP address assigned by the server
# This is the address inside the tunnel (not your public IP)
Address = 10.8.0.2/24

# IMPORTANT: Set DNS to 127.0.0.1 so all DNS goes through DNSCrypt-proxy
# Do NOT put your VPN server or any external DNS here
DNS = 127.0.0.1

[Peer]
# Your server's public key (provided by server administrator)
PublicKey = YOUR_SERVER_PUBLIC_KEY_HERE

# Optional: pre-shared key for additional security layer
# PresharedKey = YOUR_PRESHARED_KEY_HERE

# Your VPN server IP and port — note both values for pf.conf Step 5
Endpoint = YOUR_SERVER_IP:YOUR_SERVER_PORT

# Route all IPv4 traffic through the VPN tunnel
AllowedIPs = 0.0.0.0/0

# Send keepalive packet every 25 seconds — prevents NAT timeout
PersistentKeepalive = 25
```

> **Important:** `DNS = 127.0.0.1` tells WireGuard to use your local DNSCrypt-proxy for all DNS. Do not put any external DNS server here — it would bypass DNSCrypt.

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Set correct permissions — config must not be readable by other users:

```bash
# Apple Silicon
sudo chmod 600 /opt/homebrew/etc/wireguard/wg0.conf

# Intel Mac
# sudo chmod 600 /usr/local/etc/wireguard/wg0.conf
```

#### 2.3 Start WireGuard

```bash
# Start the tunnel
sudo wg-quick up wg0

# Check connection status and verify tunnel is active
sudo wg show

# Stop the tunnel
sudo wg-quick down wg0
```

---

### Step 3 — Install DNSCrypt-proxy v2.1.15

```bash
brew install dnscrypt-proxy
```

Verify the version:

```bash
dnscrypt-proxy --version
# Expected output: 2.1.15
```

---

### Step 4 — Configure DNSCrypt-proxy

#### 4.1 Copy the config file

```bash
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
```

#### 4.2 Choose your DNS servers and relays

Open https://dnscrypt.info/public-servers in your browser.

**Requirements for choosing a server:**
- Protocol: `DNSCrypt` (stronger authentication than DoH)
- `No logs` = ✓ (server does not keep query logs)
- `DNSSEC` = ✓ (server validates DNS signatures)
- Choose a server geographically close to you for lower latency

**Requirements for choosing relays:**
- Must be operated by a **different organization** than your server
- Should be in a **different country** than your server
- Example: server in Germany → relay in Netherlands or Sweden

**How to find relay names:**
Download the relay list: https://download.dnscrypt.info/resolvers-list/v3/relays.md

Pick 2-3 relay names — the identifier at the start of each entry.

#### 4.3 Edit dnscrypt-proxy.toml

```bash
nano /opt/homebrew/etc/dnscrypt-proxy.toml
```

Find the `[anonymized_dns]` section and replace placeholder values:

```toml
routes = [
    { server_name='YOUR_SERVER_NAME', via=['YOUR_RELAY_1', 'YOUR_RELAY_2'] },
]
```

#### 4.4 Generate encryption key for log privacy

```bash
openssl rand -hex 32
```

Copy the output (64 hex characters) and paste into `dnscrypt-proxy.toml`:

```toml
[ip_encryption]
algorithm = "ipcrypt-ndx"
key = "PASTE_YOUR_64_CHARACTER_KEY_HERE"
```

> **Important:** Never share or publish this key. It encrypts IP addresses in your local log files.

#### 4.5 Copy blocklist files

```bash
mkdir -p /opt/homebrew/etc/dnscrypt-proxy
cp dnscrypt/blocked-names.txt /opt/homebrew/etc/dnscrypt-proxy/blocked-names.txt
cp dnscrypt/blocked-ips.txt   /opt/homebrew/etc/dnscrypt-proxy/blocked-ips.txt
```

#### 4.6 Start DNSCrypt-proxy

```bash
brew services start dnscrypt-proxy
```

Verify it is running:

```bash
brew services list | grep dnscrypt
# Expected: dnscrypt-proxy  started
```

Test DNS resolution:

```bash
dig google.com @127.0.0.1
# Should return an answer within 1 second
```

#### 4.7 Set system DNS to 127.0.0.1

Go to: **System Settings → Network → Wi-Fi → Details → DNS tab**

1. Click **+** to add a new DNS server
2. Type `127.0.0.1`
3. Remove all other DNS servers from the list
4. Click **OK** → **Apply**

All DNS queries from your Mac will now go through DNSCrypt-proxy.

---

### Step 5 — Configure the PF Firewall

#### 5.1 How macOS PF works with a custom anchor

macOS has its own built-in PF firewall in `/etc/pf.conf`. **We do not replace this file.** Instead, we add our rules as a **custom anchor** — a separate rule set loaded alongside the Apple system rules.

```
/etc/pf.conf  (Apple system file — we add 2 lines here, nothing else)
     │
     ├── Apple system rules (NAT, scrub, etc.)
     │
     ├── anchor "custom"  ← our anchor, runs FIRST
     │        └── loads from /etc/pf.anchors/custom  ← our pf.conf goes here
     │
     └── anchor "com.apple/*"  ← Apple filters, run after ours
```

Our custom rules work as an **addition** to the Apple firewall, not a replacement. The kill-switch and DNS rules run first and take priority.

#### 5.2 Find your network values

Open your WireGuard config file:
- Apple Silicon: `/opt/homebrew/etc/wireguard/wg0.conf`
- Intel: `/usr/local/etc/wireguard/wg0.conf`

You need these 4 values:

**A) WireGuard server IP and port (`wg_endpoints`)**

```ini
[Peer]
Endpoint = 1.2.3.4:51820
           ↑       ↑
           IP      Port  ← note both values
```

**B) Bootstrap DNS (`bootstrap_dns`)**

A plain DNS server used **only** to establish the WireGuard connection before the tunnel is up. Once tunnel is active, all DNS goes through DNSCrypt on 127.0.0.1.

Choose any public DNS:
- `1.1.1.1` (Cloudflare)
- `9.9.9.9` (Quad9)
- `8.8.8.8` (Google)

**C) DNSCrypt upstream server IP (`dnscrypt_upstream`)**

The IP address of the DNSCrypt server you chose in Step 4:

```bash
# Replace with your actual server hostname from dnscrypt.info
dig +short YOUR_DNSCRYPT_SERVER_HOSTNAME
```

**D) WireGuard internal subnet (`wg_internal`)**

```ini
[Interface]
Address = 10.8.0.2/24
          ↑
          Your internal WireGuard IP — replace last number with 0
```

Examples:
- `10.8.0.2/24` → use `10.8.0.0/24`
- `10.0.0.5/24` → use `10.0.0.0/24`
- `172.16.0.2/24` → use `172.16.0.0/24`

#### 5.3 Edit pf/pf.conf

Open `pf/pf.conf` from this repository and fill in the tables:

```sh
# Value A — WireGuard server IP
table <wg_endpoints> persist { YOUR_WG_SERVER_IP }

# Value B — bootstrap DNS
table <bootstrap_dns> persist { YOUR_BOOTSTRAP_DNS_IP }

# Value C — DNSCrypt server IP
table <dnscrypt_upstream> persist { YOUR_DNSCRYPT_SERVER_IP }

# Value D — WireGuard internal subnet
table <wg_internal> persist { YOUR_WG_INTERNAL_SUBNET }
```

Update WireGuard port (from value A):

```sh
# Find this line and update the port number:
pass out log quick on $wifi proto udp to <wg_endpoints> port { YOUR_WG_PORT } keep state

# Example with default WireGuard port 51820:
# pass out log quick on $wifi proto udp to <wg_endpoints> port 51820 keep state
```

#### 5.4 Add custom anchor to system pf.conf

Check if already present:

```bash
grep -n "custom" /etc/pf.conf
```

If you see `anchor "custom"` and `load anchor "custom"` — already configured, skip to 5.5.

If **not found** — add it now:

```bash
# Backup first
sudo cp /etc/pf.conf /etc/pf.conf.backup

# Open system file
sudo nano /etc/pf.conf
```

Find the filtering section:

```sh
##### (5) FILTERING
anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
```

Add custom anchor **before** `anchor "com.apple/*"`:

```sh
##### (5) FILTERING
# Custom anchor — runs FIRST, before Apple filters
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"

# Apple system filters — after custom
anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

#### 5.5 Install custom rules

```bash
sudo cp pf/pf.conf /etc/pf.anchors/custom
```

#### 5.6 Check syntax before loading

```bash
sudo pfctl -n -f /etc/pf.conf
# No output = no errors, safe to proceed
```

#### 5.7 Load and enable PF

```bash
sudo pfctl -f /etc/pf.conf     # Load rules
sudo pfctl -e                   # Enable PF

# Verify custom anchor is loaded
sudo pfctl -a custom -sr | head -20
```

---

### Step 6 — Install PF Autostart (LaunchDaemon)

Without this, your custom PF rules are lost after every reboot.

```bash
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist
```

Verify it loaded:

```bash
sudo launchctl list | grep pfctl
# Should show a line with com.user.pfctl.enable

cat /var/log/pf_autoload.log
# Should show successful pfctl execution
```

---

### Step 7 — Connect and Verify

Start WireGuard:

```bash
sudo wg-quick up wg0
```

Run the leak verification script:

```bash
bash scripts/check-leaks.sh
```

Verify manually in your browser:
- **https://dnsleaktest.com** → Extended test → only your DNSCrypt resolver
- **https://ipleak.net** → only your VPN server IP
- **https://browserleaks.com/webrtc** → no WebRTC IP leak

#### Test the kill-switch

```bash
# Step 1: Check public IP — must show VPN server IP
curl https://ifconfig.me

# Step 2: Stop WireGuard
sudo wg-quick down wg0

# Step 3: Internet must NOT work
curl --max-time 5 https://ifconfig.me
# Expected: connection timeout

# Step 4: Reconnect
sudo wg-quick up wg0
# Internet returns immediately
```

If traffic stops when WireGuard is down — kill-switch is working correctly.

---

### Troubleshooting

#### Internet not working after loading PF rules

```bash
# 1. Check for syntax errors
sudo pfctl -n -f /etc/pf.conf
# If errors shown — fix them before reloading

# 2. Emergency — disable PF completely
sudo pfctl -d

# 3. Check which rules are actually loaded
sudo pfctl -a custom -sr | head -30

# 4. Check PF logs for blocked traffic
sudo tcpdump -n -i pflog0
```

#### DNS not resolving

```bash
# 1. Check DNSCrypt is running
brew services list | grep dnscrypt

# 2. Restart DNSCrypt
brew services restart dnscrypt-proxy

# 3. Test DNS directly
dig google.com @127.0.0.1

# 4. Check DNSCrypt log for errors
tail -50 /opt/homebrew/etc/dnscrypt-proxy.log

# 5. Verify system DNS is set to 127.0.0.1
scutil --dns | grep nameserver
```

#### WireGuard not connecting

```bash
# 1. Check tunnel status
sudo wg show

# 2. Verify server IP and port are correct in pf.conf
grep wg_endpoints /etc/pf.anchors/custom

# 3. Check if WireGuard handshake packets are allowed
sudo pfctl -ss | grep utun

# 4. Try stopping and starting the tunnel
sudo wg-quick down wg0
sudo wg-quick up wg0
```

#### PF rules lost after reboot

```bash
# 1. Check if LaunchDaemon is loaded
sudo launchctl list | grep pfctl

# 2. Check boot log for errors
cat /var/log/pf_autoload_error.log

# 3. Reload LaunchDaemon manually
sudo launchctl unload /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist
```

---

### Optional: DNS through Tor

The PF firewall **does not block Tor**. Rules for Tor loopback ports are already included in `pf.conf` — no changes to PF needed.

```bash
# 1. Install and start Tor
brew install tor
brew services start tor

# 2. Edit dnscrypt-proxy.toml — uncomment this line:
nano /opt/homebrew/etc/dnscrypt-proxy.toml
# proxy = 'socks5://dnscrypt:dnscrypt@127.0.0.1:9050'

# 3. Restart DNSCrypt
brew services restart dnscrypt-proxy

# 4. Verify DNS still resolves
dig google.com @127.0.0.1
```

> **Note:** Adds ~200-500ms latency per DNS query. Use when you need to hide DNS traffic from relay servers as well.

---

### Useful Commands

```bash
# ── WireGuard ─────────────────────────────────────────
sudo wg-quick up wg0                   # Start VPN tunnel
sudo wg-quick down wg0                 # Stop VPN tunnel
sudo wg show                           # Show tunnel status and stats

# ── PF Firewall ───────────────────────────────────────
sudo pfctl -e                          # Enable PF
sudo pfctl -d                          # Disable PF (emergency)
sudo pfctl -f /etc/pf.conf             # Reload all rules
sudo pfctl -n -f /etc/pf.conf         # Check syntax without loading
sudo pfctl -a custom -sr               # Show active custom anchor rules
sudo pfctl -ss                         # Show all active connections
sudo pfctl -ss | grep utun             # Show VPN tunnel connections only
sudo pfctl -t bogons -T show           # Show bogon IP table
sudo pfctl -t abusers -T show          # Show IPs blocked by rate-limiting
sudo pfctl -t abusers -T flush         # Clear abusers table
sudo pfctl -s info                     # Show PF statistics

# ── DNSCrypt-proxy ────────────────────────────────────
brew services start dnscrypt-proxy     # Start
brew services stop dnscrypt-proxy      # Stop
brew services restart dnscrypt-proxy   # Restart
brew services list | grep dnscrypt     # Check status
dig google.com @127.0.0.1             # Test DNS resolution
dig +short TXT whoami.ds.akahelp.net  # Check which resolver answers

# ── Logs ──────────────────────────────────────────────
cat /var/log/pf_autoload.log                     # PF boot log
cat /var/log/pf_autoload_error.log               # PF boot errors
tail -f /opt/homebrew/etc/dnscrypt-proxy.log     # DNSCrypt log

# ── Diagnostics ───────────────────────────────────────
curl https://ifconfig.me                         # Check public IP
sudo tcpdump -n -i en0 port 53                  # DNS on Wi-Fi (should be empty)
sudo tcpdump -n -i lo0 port 53                  # DNS on loopback (active)
bash scripts/check-leaks.sh                     # Full leak verification
```

---

## Contributing

Issues and pull requests are welcome. If you find a bug or have an improvement, please open an issue first to discuss the change.

---

<a name="deutsch"></a>
# 🇩🇪 Deutsch

## Ziel

Diese Konfiguration richtet sich an **datenschutzbewusste Benutzer, Sicherheitsforscher und IT-Fachleute**, die die vollständige Kontrolle über ihren Netzwerk-Stack bevorzugen.

Ziel ist es sicherzustellen, dass **der gesamte Netzwerkverkehr immer durch einen WireGuard-VPN-Tunnel geleitet wird** — ohne jede Möglichkeit einer Datenleckage. Alle DNS-Anfragen werden über DNSCrypt-proxy v2.1.15 verschlüsselt, sodass DNS-Datenverkehr nicht im Klartext abgefangen oder überwacht werden kann.

## Schutz vor

| Bedrohung | Schutz |
|-----------|--------|
| Datenleck außerhalb VPN | Kill-Switch blockiert gesamten Nicht-Tunnel-Verkehr |
| DNS-Leak | DNS auf allen Interfaces außer 127.0.0.1 gesperrt |
| DNS-Abfang | Alle DNS-Anfragen via DNSCrypt verschlüsselt |
| DNS-Hijacking | DNSCrypt nur mit Whitelist-Upstreams |
| Resolver sieht echte IP | Anonymized DNS über Relay-Server |
| IPv6-Leak | IPv6 global blockiert |
| TCP-Port-Scanning | Ungültige TCP-Flag-Kombinationen blockiert |

## Installationsübersicht

```
Schritt 1: Homebrew installieren
    ↓
Schritt 2: WireGuard installieren → Schlüssel generieren → Konfiguration erstellen
    ↓
Schritt 3: DNSCrypt-proxy installieren
    ↓
Schritt 4: DNSCrypt-proxy konfigurieren → Server wählen → Schlüssel generieren
    ↓
Schritt 5: PF-Firewall konfigurieren → IP-Tabellen ausfüllen → Custom Anchor hinzufügen
    ↓
Schritt 6: LaunchDaemon installieren (PF-Autostart beim Boot)
    ↓
Schritt 7: WireGuard verbinden → Leaks prüfen
```

## Voraussetzungen

Vor dem Start sicherstellen:

- macOS 26.4 oder neuer
- Laufender WireGuard-**Server** mit:
  - ✓ Öffentlichem Schlüssel des Servers
  - ✓ IP-Adresse und Port des Servers
  - ✓ Internem VPN-Subnetz (z.B. `10.8.0.0/24`)
- Terminal (Programme → Dienstprogramme → Terminal)

## Vollständige Installationsanleitung

### Schritt 1 — Homebrew installieren

**Terminal** öffnen und ausführen:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Das Installationsprogramm fragt nach dem macOS-Passwort — eingeben und Enter drücken (keine Zeichen werden angezeigt, das ist normal).

Installation überprüfen:

```bash
brew --version
# Erwartete Ausgabe: Homebrew 4.x.x
```

### Schritt 2 — WireGuard installieren

```bash
brew install wireguard-tools wireguard-go
```

#### 2.1 WireGuard-Schlüsselpaar generieren

Vor dem Erstellen der Konfigurationsdatei ein Schlüsselpaar generieren:

```bash
# Schlüssel generieren und in temporäre Dateien speichern
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey | wg pubkey > /opt/homebrew/etc/wireguard/publickey

# Privaten Schlüssel anzeigen — kopieren für PrivateKey =
cat /opt/homebrew/etc/wireguard/privatekey

# Öffentlichen Schlüssel anzeigen — an Server-Administrator senden
cat /opt/homebrew/etc/wireguard/publickey

# Schlüsseldateien nach dem Kopieren löschen
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey
```

> **Wichtig:** Den privaten Schlüssel niemals teilen. Bei Verlust neues Schlüsselpaar generieren.

#### 2.2 WireGuard-Konfigurationsdatei erstellen

Pfad je nach Mac-Chip:

| Mac-Chip | Konfigurationspfad |
|----------|-------------------|
| **Apple Silicon (M1/M2/M3/M4)** | `/opt/homebrew/etc/wireguard/` |
| Intel | `/usr/local/etc/wireguard/` |

> **Hinweis:** Chip prüfen: `which wg-quick` → `/opt/homebrew/...` = Apple Silicon, `/usr/local/...` = Intel

```bash
# Apple Silicon (M1/M2/M3/M4)
sudo mkdir -p /opt/homebrew/etc/wireguard
sudo nano /opt/homebrew/etc/wireguard/wg0.conf

# Intel Mac:
# sudo mkdir -p /usr/local/etc/wireguard
# sudo nano /usr/local/etc/wireguard/wg0.conf
```

Konfigurationsstruktur:

```ini
[Interface]
# Privater Schlüssel aus Schritt 2.1
PrivateKey = IHR_PRIVATER_SCHLÜSSEL

# Interne VPN-IP-Adresse (vom Server zugewiesen)
Address = 10.8.0.2/24

# WICHTIG: DNS auf 127.0.0.1 setzen — alle DNS-Anfragen gehen durch DNSCrypt
# Keine externe DNS-Adresse hier eintragen
DNS = 127.0.0.1

[Peer]
# Öffentlicher Schlüssel des Servers
PublicKey = ÖFFENTLICHER_SCHLÜSSEL_DES_SERVERS

# Optional: Pre-shared Key für zusätzliche Sicherheit
# PresharedKey = IHR_PRESHARED_KEY

# IP-Adresse und Port des VPN-Servers — beide Werte für pf.conf notieren
Endpoint = SERVER_IP:SERVER_PORT

# Gesamten IPv4-Verkehr durch VPN tunneln
AllowedIPs = 0.0.0.0/0

# Keepalive-Paket alle 25 Sekunden — verhindert NAT-Timeout
PersistentKeepalive = 25
```

> **Wichtig:** `DNS = 127.0.0.1` leitet alle DNS-Anfragen an den lokalen DNSCrypt-proxy. Keine externe DNS-Adresse hier eintragen.

Speichern: `Strg+O` → `Enter` → `Strg+X`

Berechtigungen setzen:

```bash
# Apple Silicon
sudo chmod 600 /opt/homebrew/etc/wireguard/wg0.conf
# Intel: sudo chmod 600 /usr/local/etc/wireguard/wg0.conf
```

#### 2.3 WireGuard starten

```bash
sudo wg-quick up wg0    # Tunnel starten
sudo wg show            # Verbindungsstatus prüfen
sudo wg-quick down wg0  # Tunnel stoppen
```

### Schritt 3 — DNSCrypt-proxy v2.1.15 installieren

```bash
brew install dnscrypt-proxy
```

Version überprüfen:

```bash
dnscrypt-proxy --version
# Erwartete Ausgabe: 2.1.15
```

### Schritt 4 — DNSCrypt-proxy konfigurieren

#### 4.1 Konfigurationsdatei kopieren

```bash
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
```

#### 4.2 Server und Relays auswählen

Verfügbare Server: https://dnscrypt.info/public-servers

**Serveranforderungen:**
- Protokoll: `DNSCrypt`
- `No logs` = ✓
- `DNSSEC` = ✓
- Geographisch nahen Server wählen

**Relay-Anforderungen:**
- Andere Organisation als der Server
- Anderes Land als der Server
- Beispiel: Server in Deutschland → Relay in Niederlanden oder Schweden

#### 4.3 Konfigurationsdatei bearbeiten

```bash
nano /opt/homebrew/etc/dnscrypt-proxy.toml
```

Abschnitt `[anonymized_dns]` anpassen:

```toml
routes = [
    { server_name='IHR_SERVER', via=['IHR_RELAY_1', 'IHR_RELAY_2'] },
]
```

#### 4.4 Verschlüsselungsschlüssel generieren

```bash
openssl rand -hex 32
```

Ausgabe in `dnscrypt-proxy.toml` einfügen:

```toml
[ip_encryption]
algorithm = "ipcrypt-ndx"
key = "IHR_64_ZEICHEN_SCHLÜSSEL"
```

> **Wichtig:** Diesen Schlüssel niemals veröffentlichen. Er verschlüsselt IP-Adressen in Log-Dateien.

#### 4.5 Blocklisten kopieren

```bash
mkdir -p /opt/homebrew/etc/dnscrypt-proxy
cp dnscrypt/blocked-names.txt /opt/homebrew/etc/dnscrypt-proxy/blocked-names.txt
cp dnscrypt/blocked-ips.txt   /opt/homebrew/etc/dnscrypt-proxy/blocked-ips.txt
```

#### 4.6 DNSCrypt-proxy starten und testen

```bash
brew services start dnscrypt-proxy
brew services list | grep dnscrypt   # Status prüfen
dig google.com @127.0.0.1            # DNS testen
```

#### 4.7 System-DNS auf 127.0.0.1 setzen

**Systemeinstellungen → Netzwerk → WLAN → Details → DNS-Tab**

1. **+** klicken
2. `127.0.0.1` eingeben
3. Alle anderen DNS-Server entfernen
4. **OK** → **Anwenden**

### Schritt 5 — PF-Firewall konfigurieren

#### 5.1 Struktur — Custom Anchor als Ergänzung

```
/etc/pf.conf  (Apple-Systemdatei — nur 2 Zeilen hinzufügen)
     │
     ├── Apple-Systemregeln (NAT, scrub usw.)
     │
     ├── anchor "custom"  ← unsere Regeln, laufen ZUERST
     │        └── lädt /etc/pf.anchors/custom
     │
     └── anchor "com.apple/*"  ← Apple-Filter, danach
```

#### 5.2 Netzwerkwerte ermitteln

Aus der WireGuard-Konfiguration (`/opt/homebrew/etc/wireguard/wg0.conf`):

**A) Server-IP und Port (`wg_endpoints`)**
```ini
[Peer]
Endpoint = 1.2.3.4:51820
           ↑       ↑
           IP      Port  ← beide Werte notieren
```

**B) Bootstrap-DNS (`bootstrap_dns`)** — z.B. `1.1.1.1` oder `9.9.9.9`

**C) DNSCrypt-Server-IP (`dnscrypt_upstream`)**
```bash
dig +short IHR_DNSCRYPT_SERVER_HOSTNAME
```

**D) WireGuard-internes Subnetz (`wg_internal`)**
```ini
Address = 10.8.0.2/24  →  Subnetz: 10.8.0.0/24
```
Letzte Zahl durch 0 ersetzen:
- `10.8.0.2/24` → `10.8.0.0/24`
- `10.0.0.5/24` → `10.0.0.0/24`

#### 5.3 pf/pf.conf bearbeiten

```sh
table <wg_endpoints>      persist { IHRE_SERVER_IP }
table <bootstrap_dns>     persist { 1.1.1.1 }
table <dnscrypt_upstream> persist { IHRE_DNSCRYPT_IP }
table <wg_internal>       persist { IHR_WG_SUBNETZ }
```

Port anpassen:
```sh
pass out log quick on $wifi proto udp to <wg_endpoints> port { IHR_PORT } keep state
```

#### 5.4 Custom Anchor zur Systemdatei hinzufügen

Prüfen ob bereits vorhanden:
```bash
grep -n "custom" /etc/pf.conf
```

Falls nicht vorhanden:
```bash
sudo cp /etc/pf.conf /etc/pf.conf.backup
sudo nano /etc/pf.conf
```

Vor `anchor "com.apple/*"` einfügen:
```sh
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"
```

#### 5.5 Regeln installieren und laden

```bash
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf  # Syntax prüfen (keine Ausgabe = keine Fehler)
sudo pfctl -f /etc/pf.conf     # Regeln laden
sudo pfctl -e                   # PF aktivieren
```

### Schritt 6 — Autostart installieren (LaunchDaemon)

```bash
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist
```

Überprüfen:
```bash
sudo launchctl list | grep pfctl
cat /var/log/pf_autoload.log
```

### Schritt 7 — Verbinden und überprüfen

```bash
sudo wg-quick up wg0
bash scripts/check-leaks.sh
```

Browser-Tests:
- **https://dnsleaktest.com** → nur DNSCrypt-Resolver sichtbar
- **https://ipleak.net** → nur VPN-Server-IP sichtbar
- **https://browserleaks.com/webrtc** → kein WebRTC-Leak

#### Kill-Switch testen

```bash
curl https://ifconfig.me       # VPN-Server-IP anzeigen
sudo wg-quick down wg0         # VPN trennen → Internet muss stoppen
curl --max-time 5 https://ifconfig.me  # Muss Timeout ergeben
sudo wg-quick up wg0           # VPN verbinden → Internet kehrt zurück
```

### Fehlerbehebung

#### Internet funktioniert nicht nach PF-Regelladung

```bash
# 1. Syntaxfehler prüfen
sudo pfctl -n -f /etc/pf.conf

# 2. Notfall — PF vollständig deaktivieren
sudo pfctl -d

# 3. Geladene Regeln prüfen
sudo pfctl -a custom -sr | head -30
```

#### DNS wird nicht aufgelöst

```bash
# 1. DNSCrypt-Status prüfen
brew services list | grep dnscrypt

# 2. DNSCrypt neu starten
brew services restart dnscrypt-proxy

# 3. DNS direkt testen
dig google.com @127.0.0.1

# 4. Log-Datei auf Fehler prüfen
tail -50 /opt/homebrew/etc/dnscrypt-proxy.log
```

#### WireGuard verbindet nicht

```bash
sudo wg show                    # Tunnel-Status prüfen
sudo wg-quick down wg0 && sudo wg-quick up wg0  # Neu verbinden
sudo pfctl -ss | grep utun      # States prüfen
```

#### PF-Regeln nach Neustart verloren

```bash
sudo launchctl list | grep pfctl
cat /var/log/pf_autoload_error.log
sudo launchctl unload /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist
```

### Optional: DNS über Tor

Die PF-Firewall blockiert Tor **nicht**. Loopback-Regeln für Ports 9050, 9150, 9151 sind bereits in `pf.conf` enthalten — keine Änderungen an PF erforderlich.

```bash
brew install tor && brew services start tor
# In dnscrypt-proxy.toml diese Zeile auskommentieren:
# proxy = 'socks5://dnscrypt:dnscrypt@127.0.0.1:9050'
brew services restart dnscrypt-proxy
dig google.com @127.0.0.1  # DNS testen
```

---

<a name="russian"></a>
# 🇷🇺 Русский

## Цель

Эта конфигурация предназначена для **пользователей, заботящихся о конфиденциальности, исследователей в области безопасности и ИТ-специалистов**, которые предпочитают полностью контролировать свой сетевой стек.

Цель — гарантировать что **весь сетевой трафик всегда идёт через WireGuard VPN туннель** без малейшей возможности утечки мимо него. Все DNS запросы шифруются через DNSCrypt-proxy v2.1.15 — DNS трафик не может быть перехвачен или прочитан в открытом виде.

## От чего защищает

| Угроза | Защита |
|--------|--------|
| Утечка трафика мимо VPN | Kill-switch блокирует весь не-туннельный трафик |
| DNS утечка | DNS заблокирован на всех интерфейсах кроме 127.0.0.1 |
| Перехват DNS | Все DNS запросы зашифрованы через DNSCrypt |
| DNS hijacking | DNSCrypt только с whitelist upstream'ами |
| Резолвер видит реальный IP | Anonymized DNS маршрутизирует через relay серверы |
| Bogon IP трафик | Таблица bogons блокирует все RFC-зарезервированные диапазоны |
| TCP сканирование портов | Невалидные комбинации TCP флагов заблокированы |
| IPv6 утечка | IPv6 заблокирован на всех не-loopback интерфейсах |
| IP адреса в логах | Шифрование IP через ipcrypt-ndx |

## Обзор установки

```
Шаг 1: Установить Homebrew
    ↓
Шаг 2: Установить WireGuard → сгенерировать ключи → создать конфиг
    ↓
Шаг 3: Установить DNSCrypt-proxy
    ↓
Шаг 4: Настроить DNSCrypt-proxy → выбрать серверы → сгенерировать ключ
    ↓
Шаг 5: Настроить PF файрвол → заполнить IP таблицы → добавить custom anchor
    ↓
Шаг 6: Установить LaunchDaemon (автозапуск PF при загрузке)
    ↓
Шаг 7: Подключить WireGuard → проверить отсутствие утечек
```

## Требования

Перед началом убедитесь что у вас есть:

- macOS 26.4 или новее
- Работающий WireGuard **сервер** с:
  - ✓ Публичным ключом сервера
  - ✓ IP адресом и портом сервера
  - ✓ Внутренней VPN подсетью (например `10.8.0.0/24`)
- Терминал (Программы → Утилиты → Терминал)

## Полная инструкция по установке

### Шаг 1 — Установить Homebrew

Откройте **Терминал** и выполните:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Установщик запросит пароль macOS — введите его и нажмите Enter (символы не отображаются, это нормально). После установки выполните инструкции на экране.

Проверьте установку:

```bash
brew --version
# Ожидаемый вывод: Homebrew 4.x.x
```

### Шаг 2 — Установить WireGuard

```bash
brew install wireguard-tools wireguard-go
```

#### 2.1 Сгенерировать пару ключей WireGuard

До создания конфигурационного файла нужно сгенерировать приватный и публичный ключи:

```bash
# Сгенерировать ключи и сохранить во временные файлы
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey | wg pubkey > /opt/homebrew/etc/wireguard/publickey

# Посмотреть приватный ключ — скопировать в PrivateKey =
cat /opt/homebrew/etc/wireguard/privatekey

# Посмотреть публичный ключ — отправить администратору сервера
# Сервер должен добавить его в секцию [Peer] на своей стороне
cat /opt/homebrew/etc/wireguard/publickey

# Удалить файлы ключей после копирования — они больше не нужны
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey
```

> **Важно:** Никогда не делитесь приватным ключом. При потере — сгенерируйте новую пару.

#### 2.2 Создать конфигурационный файл WireGuard

При установке через Homebrew конфигурационные файлы хранятся по пути:

| Чип Mac | Путь к конфигурации |
|---------|---------------------|
| **Apple Silicon (M1/M2/M3/M4)** | `/opt/homebrew/etc/wireguard/` |
| Intel | `/usr/local/etc/wireguard/` |

> **Примечание:** Проверьте свой чип: `which wg-quick` → `/opt/homebrew/...` = Apple Silicon, `/usr/local/...` = Intel

```bash
# Apple Silicon (M1/M2/M3/M4)
sudo mkdir -p /opt/homebrew/etc/wireguard
sudo nano /opt/homebrew/etc/wireguard/wg0.conf

# Intel Mac:
# sudo mkdir -p /usr/local/etc/wireguard
# sudo nano /usr/local/etc/wireguard/wg0.conf
```

Структура конфигурационного файла:

```ini
[Interface]
# Приватный ключ из шага 2.1
PrivateKey = ВАШ_ПРИВАТНЫЙ_КЛЮЧ

# Внутренний VPN IP адрес (назначается сервером, не публичный IP)
Address = 10.8.0.2/24

# ВАЖНО: Укажите 127.0.0.1 чтобы все DNS запросы шли через DNSCrypt-proxy
# НЕ указывайте здесь внешние DNS серверы — это обойдёт DNSCrypt
DNS = 127.0.0.1

[Peer]
# Публичный ключ сервера (предоставляет администратор сервера)
PublicKey = ПУБЛИЧНЫЙ_КЛЮЧ_СЕРВЕРА

# Опционально: предварительно согласованный ключ для доп. безопасности
# PresharedKey = ВАШ_PRESHARED_КЛЮЧ

# IP адрес и порт VPN сервера — запомните оба значения для pf.conf
Endpoint = IP_ВАШЕГО_СЕРВЕРА:ПОРТ_СЕРВЕРА

# Весь IPv4 трафик через VPN туннель
AllowedIPs = 0.0.0.0/0

# Keepalive пакет каждые 25 секунд — предотвращает NAT таймаут
PersistentKeepalive = 25
```

> **Важно:** Строка `DNS = 127.0.0.1` направляет все DNS запросы на локальный DNSCrypt-proxy. Не указывайте здесь никакие внешние DNS серверы.

Сохраните: `Ctrl+O` → `Enter` → `Ctrl+X`

Установите правильные права доступа:

```bash
# Apple Silicon
sudo chmod 600 /opt/homebrew/etc/wireguard/wg0.conf
# Intel: sudo chmod 600 /usr/local/etc/wireguard/wg0.conf
```

#### 2.3 Запустить WireGuard

```bash
sudo wg-quick up wg0    # Запустить туннель
sudo wg show            # Проверить статус соединения
sudo wg-quick down wg0  # Остановить туннель
```

### Шаг 3 — Установить DNSCrypt-proxy v2.1.15

```bash
brew install dnscrypt-proxy
```

Проверьте версию:

```bash
dnscrypt-proxy --version
# Ожидаемый вывод: 2.1.15
```

### Шаг 4 — Настроить DNSCrypt-proxy

#### 4.1 Скопировать файл конфигурации

```bash
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
```

#### 4.2 Выбрать DNS серверы и relay

Откройте https://dnscrypt.info/public-servers в браузере.

**Требования к серверу:**
- Протокол: `DNSCrypt`
- `No logs` = ✓
- `DNSSEC` = ✓
- Выбирайте географически близкий сервер

**Требования к relay:**
- Другая организация чем у сервера
- Другая страна чем у сервера
- Пример: сервер в Германии → relay в Нидерландах или Швеции

#### 4.3 Отредактировать dnscrypt-proxy.toml

```bash
nano /opt/homebrew/etc/dnscrypt-proxy.toml
```

Найдите секцию `[anonymized_dns]` и замените заглушки:

```toml
routes = [
    { server_name='ИМЯ_ВАШЕГО_СЕРВЕРА', via=['ИМЯ_RELAY_1', 'ИМЯ_RELAY_2'] },
]
```

#### 4.4 Сгенерировать ключ шифрования

```bash
openssl rand -hex 32
```

Вставьте вывод в `dnscrypt-proxy.toml`:

```toml
[ip_encryption]
algorithm = "ipcrypt-ndx"
key = "ВСТАВЬТЕ_ВАШ_64_СИМВОЛЬНЫЙ_КЛЮЧ"
```

> **Важно:** Никогда не публикуйте этот ключ. Он шифрует IP адреса в ваших лог файлах.

#### 4.5 Скопировать файлы блок-листов

```bash
mkdir -p /opt/homebrew/etc/dnscrypt-proxy
cp dnscrypt/blocked-names.txt /opt/homebrew/etc/dnscrypt-proxy/blocked-names.txt
cp dnscrypt/blocked-ips.txt   /opt/homebrew/etc/dnscrypt-proxy/blocked-ips.txt
```

#### 4.6 Запустить и протестировать DNSCrypt-proxy

```bash
brew services start dnscrypt-proxy
brew services list | grep dnscrypt   # Проверить статус
dig google.com @127.0.0.1            # Тест DNS
```

#### 4.7 Установить системный DNS на 127.0.0.1

**Системные настройки → Сеть → Wi-Fi → Подробнее → вкладка DNS**

1. Нажмите **+**
2. Введите `127.0.0.1`
3. Удалите все остальные DNS серверы
4. Нажмите **OK** → **Применить**

### Шаг 5 — Настроить PF Firewall

#### 5.1 Структура — кастомный anchor как дополнение

```
/etc/pf.conf  (системный файл Apple — добавляем только 2 строки)
     │
     ├── Системные правила Apple (NAT, scrub и т.д.)
     │
     ├── anchor "custom"  ← наши правила, выполняются ПЕРВЫМИ
     │        └── загружает /etc/pf.anchors/custom
     │
     └── anchor "com.apple/*"  ← фильтры Apple, после наших
```

#### 5.2 Найти значения для вашей сети

Откройте WireGuard конфиг (`/opt/homebrew/etc/wireguard/wg0.conf`):

**A) IP и порт сервера (`wg_endpoints`)**
```ini
[Peer]
Endpoint = 1.2.3.4:51820
           ↑       ↑
           IP      Порт  ← запомните оба значения
```

**B) Bootstrap DNS (`bootstrap_dns`)** — например `1.1.1.1` или `9.9.9.9`

**C) IP DNSCrypt сервера (`dnscrypt_upstream`)**
```bash
dig +short ВАШ_DNSCRYPT_HOSTNAME
```

**D) Внутренняя подсеть WireGuard (`wg_internal`)**
```ini
Address = 10.8.0.2/24  →  подсеть: 10.8.0.0/24
```
Замените последнее число на 0:
- `10.8.0.2/24` → `10.8.0.0/24`
- `10.0.0.5/24` → `10.0.0.0/24`

#### 5.3 Отредактировать pf/pf.conf

```sh
table <wg_endpoints>      persist { ВАШ_WG_SERVER_IP }
table <bootstrap_dns>     persist { 1.1.1.1 }
table <dnscrypt_upstream> persist { ВАШ_DNSCRYPT_SERVER_IP }
table <wg_internal>       persist { ВАШ_WG_SUBNET }
```

Обновите порт:
```sh
pass out log quick on $wifi proto udp to <wg_endpoints> port { ВАШ_ПОРТ } keep state
```

#### 5.4 Добавить кастомный anchor в системный pf.conf

Проверьте наличие:
```bash
grep -n "custom" /etc/pf.conf
```

Если не найдено:
```bash
sudo cp /etc/pf.conf /etc/pf.conf.backup
sudo nano /etc/pf.conf
```

Добавьте перед `anchor "com.apple/*"`:
```sh
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"
```

#### 5.5 Установить и загрузить правила

```bash
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf  # Проверить синтаксис (нет вывода = нет ошибок)
sudo pfctl -f /etc/pf.conf     # Загрузить правила
sudo pfctl -e                   # Включить PF
```

### Шаг 6 — Установить автозапуск (LaunchDaemon)

Без этого правила PF сбрасываются при каждой перезагрузке.

```bash
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist
```

Проверьте:
```bash
sudo launchctl list | grep pfctl
cat /var/log/pf_autoload.log
```

### Шаг 7 — Подключить и проверить

```bash
sudo wg-quick up wg0
bash scripts/check-leaks.sh
```

Проверьте в браузере:
- **https://dnsleaktest.com** → только ваш DNSCrypt резолвер
- **https://ipleak.net** → только IP вашего VPN сервера
- **https://browserleaks.com/webrtc** → нет WebRTC утечки

#### Тест kill-switch

```bash
curl https://ifconfig.me              # IP VPN сервера
sudo wg-quick down wg0                # Отключить VPN → интернет должен пропасть
curl --max-time 5 https://ifconfig.me # Должен вернуть таймаут
sudo wg-quick up wg0                  # Подключить → интернет возвращается
```

### Устранение неполадок

#### Интернет не работает после загрузки правил PF

```bash
# 1. Проверить синтаксис
sudo pfctl -n -f /etc/pf.conf

# 2. Аварийное отключение PF
sudo pfctl -d

# 3. Посмотреть загруженные правила
sudo pfctl -a custom -sr | head -30
```

#### DNS не резолвится

```bash
# 1. Проверить статус DNSCrypt
brew services list | grep dnscrypt

# 2. Перезапустить DNSCrypt
brew services restart dnscrypt-proxy

# 3. Тест DNS напрямую
dig google.com @127.0.0.1

# 4. Проверить лог на ошибки
tail -50 /opt/homebrew/etc/dnscrypt-proxy.log
```

#### WireGuard не подключается

```bash
sudo wg show                    # Проверить статус туннеля
sudo wg-quick down wg0 && sudo wg-quick up wg0  # Переподключить
sudo pfctl -ss | grep utun      # Проверить state'ы
```

#### Правила PF сброшены после перезагрузки

```bash
sudo launchctl list | grep pfctl
cat /var/log/pf_autoload_error.log
sudo launchctl unload /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist
```

### Опционально: DNS через Tor

PF файрвол **не блокирует Tor**. Правила для loopback портов Tor уже включены в `pf.conf` — изменений в PF не требуется.

```bash
brew install tor && brew services start tor
# В dnscrypt-proxy.toml раскомментируйте эту строку:
# proxy = 'socks5://dnscrypt:dnscrypt@127.0.0.1:9050'
brew services restart dnscrypt-proxy
dig google.com @127.0.0.1  # Проверить что DNS работает
```

---

## License

MIT — free to use, modify, and distribute.

## Contributing

Issues and pull requests are welcome. If you find a bug or have an improvement, please open an issue first to discuss the change.

---

*Tested on: macOS 26.4 · DNSCrypt-proxy v2.1.15 · wireguard-tools · wireguard-go*
