# macOS PF Firewall · WireGuard Kill-Switch · DNSCrypt

A macOS setup that forces all traffic through WireGuard VPN, prevents leaks
using PF firewall, and encrypts DNS queries via DNSCrypt-proxy.
Built on macOS-native tools, configured as plain text files, automated at boot.

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

## Why I Built This

Most VPN apps work fine under normal conditions — but two behaviors bothered me.

First, when a VPN connection drops unexpectedly, traffic falls back to the
regular network with no warning. Second, DNS queries often bypass the VPN
tunnel entirely, even while connected.

I wanted to understand how macOS PF firewall actually works and use it to
address both gaps: a firewall-level kill-switch that stops all traffic when
the tunnel drops, and encrypted DNS that stays on loopback regardless of
connection state.

This project is the result of working through that from first principles.

---

## Who This Is For

- Sysadmins and network engineers interested in macOS firewall configuration
- Developers who want to understand how PF rules, DNS resolution, and VPN
  routing interact in practice
- Users who want more predictable VPN behavior than most apps provide
- Anyone who prefers reading config files over trusting a black-box app

**This is not a plug-and-play consumer tool.** It requires a self-hosted
WireGuard server, terminal access, and comfort editing configuration files.

---

## Real-World Context

This setup is most useful when:

- You regularly use networks you do not control (hotel Wi-Fi, conferences,
  coworking spaces) and want consistent, predictable behavior
- You are testing or learning PF firewall rule behavior on macOS
- You want to understand how DNS resolution, VPN tunneling, and firewall
  policy interact at the OS level
- You want the network to behave the same way every time, with no silent
  fallbacks

It is also a practical learning project — PF rule syntax, DNS flow,
and LaunchDaemon configuration are all documented inline and editable.

---

## How It Works

Three things happen in order when your Mac sends a network request:

**1. DNS is handled locally**
All DNS queries go to `127.0.0.1:53`. DNSCrypt-proxy receives them, encrypts
them, and forwards through a relay server to an upstream resolver. The relay
sees your IP but not the query. The resolver sees the query but not your IP.

**2. PF enforces the tunnel**
The firewall blocks all outbound traffic on physical interfaces except what
is needed to bring up WireGuard: DHCP, a bootstrap DNS server, and the
WireGuard handshake. Everything else must go through the `utun*` interface.

**3. WireGuard carries the rest**
Once the tunnel is up, all TCP, UDP, and ICMP traffic flows through the
encrypted WireGuard tunnel to your VPN server.

If WireGuard drops, PF has no pass rule for the physical interface —
traffic stops until the tunnel is restored. No silent fallback.

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

## What This Improves

Observable, practical outcomes:

- **Traffic stops when the tunnel drops.** PF blocks all outbound traffic
  on physical interfaces when WireGuard is not active. No fallback routing.
- **DNS queries consistently go through the local resolver.** All DNS goes
  through DNSCrypt-proxy on `127.0.0.1`. The firewall enforces this —
  apps cannot override it.
- **Network behavior is inspectable.** Rules are plain text files.
  Run `sudo pfctl -a custom -sr` at any time to see exactly what is active.
- **Rules load automatically at boot.** A LaunchDaemon handles this with
  a short delay for interface initialization. Changes to `pf.conf` trigger
  an automatic reload via `WatchPaths`.

**What this does not change:** the VPN server operator can still see your
traffic destinations. Browser fingerprinting and cookie-based tracking are
unaffected. This is a network-level policy, not an endpoint security solution.

---

## Trade-offs

These are engineering decisions with real costs — worth knowing before applying:

| Decision | Effect |
|----------|--------|
| Block UDP/443 (QUIC) | HTTP/3 disabled — browsers fall back to HTTP/2 automatically |
| Disable IPv6 globally | Prevents tunnel bypass; breaks IPv6-only services |
| DNS only via loopback | Adds a few milliseconds of DNS latency |
| Anonymized DNS relay | One extra network hop — slightly higher DNS latency |
| AirDrop / mDNS blocked | Local network discovery does not work while active |

None of these affect typical daily use, but they are worth understanding
before deploying.

---

## What This Setup Addresses

| Behavior | How it is handled |
|----------|------------------|
| Traffic leaving when VPN drops | PF blocks all physical interface traffic without an active tunnel |
| DNS visible to ISP | DNS locked to loopback, encrypted via DNSCrypt |
| DNS resolver seeing client IP | Anonymized relay routing — resolver and relay each see only part |
| DNS hijacking | Whitelist-only upstream resolver configuration |
| Bogon/reserved IP routing | Full RFC bogon table on all interfaces |
| TCP port scanning | Invalid TCP flag combinations blocked |
| QUIC bypassing DNS controls | UDP 443/853 blocked inside the tunnel |
| IPv6 bypassing the tunnel | IPv6 blocked on all non-loopback interfaces |

**Out of scope:**
- Compromised VPN server
- Malware with root access (can modify PF rules)
- Browser fingerprinting or account-based tracking

---

## Technologies Used

| Technology | Role | Version |
|-----------|------|---------|
| **PF (Packet Filter)** | Stateful firewall, kill-switch, traffic policy | macOS built-in |
| **WireGuard** | VPN tunnel protocol | wireguard-tools + wireguard-go |
| **DNSCrypt-proxy** | Encrypted DNS with relay routing | v2.1.15 |
| **launchd / LaunchDaemon** | Boot persistence, WatchPaths reload | macOS built-in |
| **Homebrew** | Package management | 4.x |

---

## Repository Structure

```
macos-pf-wireguard-hardened/
├── pf/
│   └── pf.conf                        # Custom PF anchor — kill-switch + DNS rules
├── dnscrypt/
│   ├── dnscrypt-proxy.toml            # DNSCrypt-proxy configuration
│   ├── blocked-names.txt              # Domain blocklist
│   └── blocked-ips.txt                # IP blocklist
├── launchdaemons/
│   └── com.user.pfctl.enable.plist    # LaunchDaemon — boot persistence + WatchPaths
├── scripts/
│   └── check-leaks.sh                 # Automated verification script
└── README.md
```

---

## Quick Start

Minimum steps. For full detail see [Full Installation Guide](#full-installation-guide).

```bash
# 1. Install dependencies
brew install wireguard-tools wireguard-go dnscrypt-proxy

# 2. Generate WireGuard keys
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey \
          | wg pubkey > /opt/homebrew/etc/wireguard/publickey
# privatekey → paste into PrivateKey in wg-0.conf
# publickey  → send to your server admin
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey

# 3. Create WireGuard config
# Note: use wg-0.conf (not wg0.conf) — required for wg-quick on macOS via Homebrew
sudo nano /opt/homebrew/etc/wireguard/wg-0.conf
# Fill in: PrivateKey, Address, DNS=127.0.0.1, PublicKey, Endpoint, AllowedIPs=0.0.0.0/0
sudo chmod 600 /opt/homebrew/etc/wireguard/wg-0.conf

# 4. Configure DNSCrypt
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
# Edit: set routes in [anonymized_dns], generate key: openssl rand -hex 32
brew services start dnscrypt-proxy

# 5. Set system DNS to 127.0.0.1
# System Settings → Network → Wi-Fi → Details → DNS → 127.0.0.1

# 6. Fill in pf/pf.conf tables, then install and load
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf   # validate first
sudo pfctl -f /etc/pf.conf && sudo pfctl -e

# 7. Install LaunchDaemon
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist

# 8. Connect and verify
sudo wg-quick up wg-0
bash ~/scripts/check-leaks.sh
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
# Apple Silicon path
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey \
          | wg pubkey > /opt/homebrew/etc/wireguard/publickey

cat /opt/homebrew/etc/wireguard/privatekey   # → paste into PrivateKey
cat /opt/homebrew/etc/wireguard/publickey    # → send to server admin

rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey
```

> Config paths: Apple Silicon → `/opt/homebrew/etc/wireguard/` · Intel → `/usr/local/etc/wireguard/`
> Check chip: `which wg-quick` — path prefix indicates chip type

**Create tunnel config:**

> **Note:** On macOS with Homebrew, name your config `wg-0.conf` (with a hyphen),
> not `wg0.conf`. The filename without hyphen may fail to start with `wg-quick`.

```bash
sudo nano /opt/homebrew/etc/wireguard/wg-0.conf
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
sudo chmod 600 /opt/homebrew/etc/wireguard/wg-0.conf
sudo wg-quick up wg-0 && sudo wg show
```

---

### Step 3 — Install DNSCrypt-proxy

```bash
brew install dnscrypt-proxy
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
```

**Configure servers and relays** — browse https://dnscrypt.info/public-servers

Select servers with: `DNSCrypt` protocol · `No logs` · `DNSSEC`

Select relays from a **different organization** and **different country** than the server.

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

This key encrypts IP addresses in local DNSCrypt log files.
Generate it and paste it into `dnscrypt-proxy.toml`:

```bash
openssl rand -hex 32
```

Find this section in `dnscrypt-proxy.toml` and paste the output as the key value:

```toml
[ip_encryption]
  algorithm = "ipcrypt-ndx"
  key = "PASTE_YOUR_64_CHARACTER_HEX_KEY_HERE"
```

> Keep this key private — do not commit it to a public repository.

**Copy blocklists and start:**

```bash
mkdir -p /opt/homebrew/etc/dnscrypt-proxy
cp dnscrypt/blocked-names.txt /opt/homebrew/etc/dnscrypt-proxy/
cp dnscrypt/blocked-ips.txt   /opt/homebrew/etc/dnscrypt-proxy/

brew services start dnscrypt-proxy
dig google.com @127.0.0.1
```

**Set system DNS:** System Settings → Network → Wi-Fi → Details → DNS → `127.0.0.1`, remove all others.

---

### Step 4 — Configure PF Firewall

**What the rules do:**
- Default: block all traffic in both directions
- Before tunnel: allow DHCP, bootstrap DNS, and WireGuard handshake only
- Kill-switch: block all outbound on physical interfaces after those exceptions
- Inside tunnel: allow traffic through `utun*` with rate limiting and bogon filtering

> **Two separate files:**
>
> `/etc/pf.conf` — macOS system file. Add two lines here only (anchor reference).
> Do not put IP addresses here.
>
> `/etc/pf.anchors/custom` — your ruleset. Fill in all tables and rules here.

#### 4.1 — Edit your custom rules file

Open `pf/pf.conf` from this repository and fill in the four tables:

```sh
# WireGuard server IP — from [Peer] → Endpoint in wg-0.conf
table <wg_endpoints>      persist { YOUR_WG_SERVER_IP }

# Plain DNS used only before the tunnel is up
# Once tunnel is active, all DNS goes through DNSCrypt on 127.0.0.1
table <bootstrap_dns>     persist { 1.1.1.1 }

# IP of your DNSCrypt upstream server
# Find it: dig +short YOUR_DNSCRYPT_SERVER_HOSTNAME
# If using relay-only routing, leave this empty
table <dnscrypt_upstream> persist { YOUR_DNSCRYPT_SERVER_IP }

# WireGuard internal subnet
# From [Interface] → Address in wg-0.conf — replace last octet with 0
# Example: Address = 10.8.0.5/24 → use 10.8.0.0/24
table <wg_internal>       persist { 10.8.0.0/24 }
```

Update the WireGuard port — find it in `[Peer] → Endpoint` in `wg-0.conf`
(e.g. `1.2.3.4:51820` → port is `51820`):

```sh
# Find this line in pf/pf.conf and update the port:
pass out log quick on $wifi proto udp to <wg_endpoints> port { YOUR_PORT } keep state
```

#### 4.2 — Install your rules

```bash
sudo cp pf/pf.conf /etc/pf.anchors/custom
```

#### 4.3 — Add anchor to system pf.conf

You are adding two lines only — nothing else in this file changes.

```bash
grep -n "custom" /etc/pf.conf   # check if already present
sudo cp /etc/pf.conf /etc/pf.conf.backup
sudo nano /etc/pf.conf
```

Find the filtering section and insert **before** `anchor "com.apple/*"`:

```sh
##### (5) FILTERING

# Custom rules — run first, before Apple system filters
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"

anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

#### 4.4 — Validate and load

```bash
# Validate syntax — no output means no errors
sudo pfctl -n -f /etc/pf.conf

# Load and enable
sudo pfctl -f /etc/pf.conf && sudo pfctl -e

# Confirm rules are active
sudo pfctl -a custom -sr | head -20
```

---

### Step 5 — Install LaunchDaemon

Loads PF rules at every boot. Includes a 5-second delay for interface
initialization. `WatchPaths` triggers automatic reload on `pf.conf` changes.

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
sudo wg-quick up wg-0
bash ~/scripts/check-leaks.sh
```

Online verification:
- https://dnsleaktest.com — Extended test
- https://ipleak.net
- https://browserleaks.com/webrtc

**Kill-switch test:**

```bash
curl https://ifconfig.me               # should show VPN server IP
sudo wg-quick down wg-0               # disconnect
curl --max-time 5 https://ifconfig.me  # should timeout — no fallback
sudo wg-quick up wg-0                 # reconnect
```

---

### Leak Verification Script

`check-leaks.sh` runs 8 checks automatically after setup.

```bash
bash ~/scripts/check-leaks.sh

# Skip the kill-switch test (briefly disconnects WireGuard):
bash ~/scripts/check-leaks.sh --no-killswitch
```

**Checks:**

1. WireGuard tunnel active, interface found, handshake recent
2. PF enabled, custom anchor loaded with rules
3. Kill-switch and IPv6 block rules present
4. DNSCrypt-proxy running, DNS resolves via `127.0.0.1`
5. System DNS resolvers are loopback-only
6. DNS resolver identity — what IP the upstream resolver sees
7. Current public IP address
8. Kill-switch test — disconnects WireGuard briefly, confirms traffic stops

**Expected output when configured correctly:**

```
╔══════════════════════════════════════════════╗
║     macOS Network Leak Verification          ║
║     PF · WireGuard · DNSCrypt                ║
╚══════════════════════════════════════════════╝

── 1. WireGuard Tunnel ──────────────────────────────────────────
  ✓ WireGuard active on interface: utun4
  ✓ Tunnel IP: 10.8.0.x
  ✓ Last handshake: 42s ago (fresh)
  → Config found: /opt/homebrew/etc/wireguard/wg-0.conf

── 2. PF Firewall ───────────────────────────────────────────────
  ✓ PF is enabled
  ✓ Custom anchor loaded (496 rules)
  ✓ Kill-switch block rules present
  ✓ IPv6 block rules present
  ✓ DNS isolation confirmed via DNSCrypt-proxy on 127.0.0.1

── 3. DNSCrypt-proxy ────────────────────────────────────────────
  ✓ dnscrypt-proxy is running
  ✓ DNS via 127.0.0.1 resolves (google.com → x.x.x.x)
  ✓ DNSSEC validation working (invalid domain blocked)

── 4. System DNS Configuration ──────────────────────────────────
  ✓ Default DNS resolver: 127.0.0.1

── 5. DNS Resolver Identity ─────────────────────────────────────
  ✓ Resolver reports your address as: 2c0f:... (relay IPv6)
  ✓ Resolver sees relay IPv6 address — routing working correctly
  → Your tunnel IP (10.8.0.x) is not visible to the resolver

── 6. Public IP Address ─────────────────────────────────────────
  ✓ Public IP: x.x.x.x
  → Confirm this matches your VPN server IP

── 7. IPv6 Leak ─────────────────────────────────────────────────
  ✓ No routable IPv6 addresses
  ✓ No IPv6 DNS responses (AAAA correctly blocked)

── 8. Kill-Switch Test ──────────────────────────────────────────
  ✓ Kill-switch working — no traffic leaked when tunnel was down

╔══════════════════════════════════════════════╗
║                  Summary                    ║
╚══════════════════════════════════════════════╝

  Passed    8
  Warnings  0
  Failed    0

  All checks passed.
```

> **Check 5 — DNS Resolver Identity:**
> The script queries `whoami.ds.akahelp.net`, a diagnostic domain that returns
> the IP the upstream resolver sees. With relay routing active, the resolver sees
> the relay's IP — not yours. An IPv6 relay address (e.g. `2c0f:...`) is expected
> and correct.

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
sudo wg-quick up wg-0 / down wg-0      # connect / disconnect
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
dig +short TXT whoami.ds.akahelp.net   # check what IP the resolver sees

# Logs
tail -f /var/log/pf_autoload.log
tail -f /opt/homebrew/etc/dnscrypt-proxy.log
sudo tcpdump -n -i en0 port 53         # should be empty — all DNS on loopback
```

---

## Contributing

Issues and pull requests are welcome. Please open an issue before submitting
a PR to discuss the proposed change.

---

*Tested on: macOS 26.4 · M1 · DNSCrypt-proxy v2.1.15 · wireguard-tools · wireguard-go*

---

<a name="deutsch"></a>
#  Deutsch

## Warum ich das gebaut habe

Die meisten VPN-Apps funktionieren unter normalen Bedingungen gut — aber zwei Verhaltensweisen haben mich gestört.

Erstens: Wenn eine VPN-Verbindung unerwartet abbricht, fließt der Traffic ohne Warnung über das reguläre Netzwerk weiter. Zweitens: DNS-Anfragen umgehen häufig den VPN-Tunnel vollständig — auch bei aktiver Verbindung.

Ich wollte verstehen, wie der macOS PF Firewall tatsächlich funktioniert, und diese Lücken richtig schließen: ein Kill-Switch auf Firewall-Ebene der Traffic stoppt wenn der Tunnel abbricht, und verschlüsseltes DNS das unabhängig vom Verbindungsstatus auf Loopback bleibt.

Dieses Projekt ist das Ergebnis dieser Arbeit von Grund auf.

---

## Für wen das gedacht ist

- Sysadmins und Netzwerktechniker die sich für macOS Firewall-Konfiguration interessieren
- Entwickler die verstehen wollen wie PF-Regeln, DNS-Auflösung und VPN-Routing zusammenspielen
- Nutzer die zuverlässigeres VPN-Verhalten als die meisten Apps wollen
- Menschen die lieber Konfigurationsdateien lesen als einer Black-Box-App vertrauen

**Kein Plug-and-Play-Tool.** Erfordert einen selbst gehosteten WireGuard-Server, Terminal-Zugang und Komfort beim Bearbeiten von Konfigurationsdateien.

---

## Praxiskontext

Dieses Setup ist am sinnvollsten wenn:

- Sie regelmäßig Netzwerke nutzen die Sie nicht kontrollieren (Hotel-WLAN, Konferenzen, Coworking) und konsistentes Verhalten wollen
- Sie PF-Firewallregeln auf macOS testen oder erlernen
- Sie verstehen wollen wie DNS-Auflösung, VPN-Tunneling und Firewall-Policy auf OS-Ebene interagieren
- Sie möchten dass das Netzwerk sich jedes Mal gleich verhält, ohne stillen Fallback

Es ist auch ein praktisches Lernprojekt — PF-Regelsyntax, DNS-Flow und LaunchDaemon-Konfiguration sind alle inline dokumentiert und editierbar.

---

## Wie es funktioniert

**1. DNS wird lokal verarbeitet**
Alle DNS-Anfragen gehen an `127.0.0.1:53`. DNSCrypt-proxy empfängt, verschlüsselt und leitet sie über einen Relay-Server zum Upstream-Resolver. Der Relay sieht die IP, aber nicht den Query-Inhalt. Der Resolver sieht den Query, aber nicht die IP.

**2. PF erzwingt den Tunnel**
Die Firewall blockiert allen ausgehenden Traffic auf physischen Interfaces — außer dem Minimum für WireGuard: DHCP, Bootstrap-DNS und WireGuard-Handshake.

**3. WireGuard trägt den Rest**
Sobald der Tunnel aktiv ist, fließt alles — TCP, UDP, ICMP — verschlüsselt zum VPN-Server.

Bricht WireGuard ab, hat PF keine Pass-Regel für das physische Interface — Traffic stoppt bis der Tunnel wiederhergestellt ist. Kein stiller Fallback.

---

## Was sich verbessert

- **Traffic stoppt wenn der Tunnel abbricht.** PF blockiert allen ausgehenden Traffic auf physischen Interfaces solange WireGuard nicht aktiv ist.
- **DNS-Anfragen gehen konsistent durch den lokalen Resolver.** Alles DNS läuft über DNSCrypt-proxy auf `127.0.0.1`. Die Firewall erzwingt dies — Apps können es nicht überschreiben.
- **Netzwerkverhalten ist inspizierbar.** Regeln sind Textdateien. `sudo pfctl -a custom -sr` zeigt jederzeit genau was aktiv ist.
- **Regeln laden automatisch beim Boot.** Ein LaunchDaemon übernimmt das mit kurzem Delay für Interface-Initialisierung. Änderungen an `pf.conf` lösen automatisches Reload via `WatchPaths` aus.

**Was sich nicht ändert:** Der VPN-Server-Betreiber sieht weiterhin Ihre Traffic-Ziele. Browser-Fingerprinting und Cookie-Tracking sind nicht betroffen.

---

## Abwägungen

| Entscheidung | Auswirkung |
|--------------|-----------|
| UDP/443 (QUIC) blockieren | HTTP/3 deaktiviert — Browser fallen automatisch auf HTTP/2 zurück |
| IPv6 global deaktivieren | Verhindert Tunnel-Bypass; bricht IPv6-only-Dienste |
| DNS nur über Loopback | Wenige Millisekunden zusätzliche DNS-Latenz |
| Anonymized DNS Relay | Ein Netzwerk-Hop mehr — etwas höhere DNS-Latenz |
| AirDrop / mDNS blockiert | Lokale Netzwerk-Erkennung funktioniert nicht während aktiv |

---

## Was dieses Setup adressiert

| Verhalten | Wie es behandelt wird |
|-----------|----------------------|
| Traffic beim VPN-Ausfall | PF blockiert alle physischen Interfaces ohne aktiven Tunnel |
| DNS für ISP sichtbar | DNS auf Loopback gesperrt, verschlüsselt via DNSCrypt |
| Resolver sieht Client-IP | Anonymized Relay-Routing — Resolver und Relay sehen jeweils nur einen Teil |
| DNS-Hijacking | Nur Whitelist-Upstream-Konfiguration |
| Bogon-IP-Routing | Vollständige RFC-Bogon-Tabelle auf allen Interfaces |
| TCP-Port-Scanning | Ungültige TCP-Flag-Kombinationen blockiert |
| QUIC umgeht DNS-Kontrollen | UDP 443/853 im Tunnel blockiert |
| IPv6 umgeht den Tunnel | IPv6 auf allen Non-Loopback-Interfaces blockiert |

**Nicht im Scope:**
- Kompromittierter VPN-Server
- Malware mit Root-Zugriff
- Browser-Fingerprinting oder Account-basiertes Tracking

---

## Verwendete Technologien

| Technologie | Rolle | Version |
|-------------|-------|---------|
| **PF (Packet Filter)** | Stateful Firewall, Kill-Switch, Traffic-Policy | macOS built-in |
| **WireGuard** | VPN-Tunnelprotokoll | wireguard-tools + wireguard-go |
| **DNSCrypt-proxy** | Verschlüsseltes DNS mit Relay-Routing | v2.1.15 |
| **launchd / LaunchDaemon** | Boot-Persistenz, WatchPaths-Reload | macOS built-in |
| **Homebrew** | Paketverwaltung | 4.x |

---

## Quick Start

```bash
# 1. Abhängigkeiten installieren
brew install wireguard-tools wireguard-go dnscrypt-proxy

# 2. WireGuard-Schlüssel generieren
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey \
          | wg pubkey > /opt/homebrew/etc/wireguard/publickey
# privatekey → in PrivateKey von wg-0.conf einfügen
# publickey  → an Server-Admin senden
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey

# 3. WireGuard-Konfiguration erstellen
# Hinweis: wg-0.conf verwenden (mit Bindestrich), nicht wg0.conf
sudo nano /opt/homebrew/etc/wireguard/wg-0.conf
# Ausfüllen: PrivateKey, Address, DNS=127.0.0.1, PublicKey, Endpoint, AllowedIPs=0.0.0.0/0
sudo chmod 600 /opt/homebrew/etc/wireguard/wg-0.conf

# 4. DNSCrypt konfigurieren
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
# Bearbeiten: routes in [anonymized_dns] setzen, Schlüssel: openssl rand -hex 32
brew services start dnscrypt-proxy

# 5. System-DNS auf 127.0.0.1 setzen
# Systemeinstellungen → Netzwerk → WLAN → Details → DNS → 127.0.0.1

# 6. pf/pf.conf Tabellen ausfüllen, dann installieren und laden
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf   # zuerst validieren
sudo pfctl -f /etc/pf.conf && sudo pfctl -e

# 7. LaunchDaemon installieren
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist

# 8. Verbinden und prüfen
sudo wg-quick up wg-0
bash ~/scripts/check-leaks.sh
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

### Schritt 2 — WireGuard installieren und konfigurieren

```bash
brew install wireguard-tools wireguard-go
```

Schlüsselpaar generieren:

```bash
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey \
          | wg pubkey > /opt/homebrew/etc/wireguard/publickey

cat /opt/homebrew/etc/wireguard/privatekey   # → in PrivateKey einfügen
cat /opt/homebrew/etc/wireguard/publickey    # → an Server-Admin senden

rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey
```

> Pfad: Apple Silicon → `/opt/homebrew/etc/wireguard/` · Intel → `/usr/local/etc/wireguard/`
> Chip prüfen: `which wg-quick`

> **Hinweis:** Konfigurationsdatei `wg-0.conf` nennen (mit Bindestrich), nicht `wg0.conf`. Der Dateiname ohne Bindestrich kann bei `wg-quick` auf macOS mit Homebrew fehlschlagen.

```bash
sudo nano /opt/homebrew/etc/wireguard/wg-0.conf
```

```ini
[Interface]
PrivateKey = IHR_PRIVATER_SCHLÜSSEL
Address    = 10.8.0.2/24
DNS        = 127.0.0.1          # Zeigt auf DNSCrypt-proxy — nicht ändern

[Peer]
PublicKey           = ÖFFENTLICHER_SERVER_SCHLÜSSEL
Endpoint            = SERVER_IP:SERVER_PORT
AllowedIPs          = 0.0.0.0/0
PersistentKeepalive = 25
```

```bash
sudo chmod 600 /opt/homebrew/etc/wireguard/wg-0.conf
sudo wg-quick up wg-0 && sudo wg show
```

### Schritt 3 — DNSCrypt-proxy installieren

```bash
brew install dnscrypt-proxy
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
```

Server: https://dnscrypt.info/public-servers — Protokoll `DNSCrypt`, `No logs` ✓, `DNSSEC` ✓

Relays: andere Organisation + anderes Land als der Server.

```bash
nano /opt/homebrew/etc/dnscrypt-proxy.toml
```

```toml
# Abschnitt [anonymized_dns]
routes = [
    { server_name='IHR_SERVER', via=['IHR_RELAY_1', 'IHR_RELAY_2'] },
]
```

**Verschlüsselungsschlüssel für Log-Dateien generieren:**

Dieser Schlüssel verschlüsselt IP-Adressen in lokalen DNSCrypt-Log-Dateien.
Generieren und in `dnscrypt-proxy.toml` einfügen:

```bash
openssl rand -hex 32
```

Den Abschnitt in `dnscrypt-proxy.toml` finden und die Ausgabe als Schlüsselwert einfügen:

```toml
[ip_encryption]
  algorithm = "ipcrypt-ndx"
  key = "HIER_DEN_64_ZEICHEN_HEX_SCHLÜSSEL_EINFÜGEN"
```

> Diesen Schlüssel privat halten — nicht in ein öffentliches Repository committen.

Blocklisten kopieren und starten:

```bash
mkdir -p /opt/homebrew/etc/dnscrypt-proxy
cp dnscrypt/blocked-names.txt /opt/homebrew/etc/dnscrypt-proxy/
cp dnscrypt/blocked-ips.txt   /opt/homebrew/etc/dnscrypt-proxy/

brew services start dnscrypt-proxy
dig google.com @127.0.0.1
```

System-DNS: **Systemeinstellungen → Netzwerk → WLAN → Details → DNS** → `127.0.0.1`, alle anderen entfernen.

### Schritt 4 — PF-Firewall konfigurieren

> **Zwei separate Dateien:**
>
> `/etc/pf.conf` — macOS-Systemdatei. Nur zwei Zeilen hinzufügen (Anchor-Verweis). Keine IP-Adressen hier.
>
> `/etc/pf.anchors/custom` — Ihr Regelwerk. Alle Tabellen und Regeln hier ausfüllen.

#### 4.1 — Eigene Regeldatei bearbeiten

`pf/pf.conf` öffnen und die vier Tabellen ausfüllen:

```sh
# WireGuard-Server-IP — aus [Peer] → Endpoint in wg-0.conf
table <wg_endpoints>      persist { IHRE_WG_SERVER_IP }

# Normaler DNS nur VOR dem Tunnel
# Sobald Tunnel aktiv: alles DNS über DNSCrypt auf 127.0.0.1
table <bootstrap_dns>     persist { 1.1.1.1 }

# IP des DNSCrypt-Upstream-Servers
# Ermitteln: dig +short IHR_DNSCRYPT_SERVER_HOSTNAME
# Bei Relay-only-Routing: leer lassen
table <dnscrypt_upstream> persist { IHRE_DNSCRYPT_SERVER_IP }

# WireGuard internes Subnetz
# Aus [Interface] → Address in wg-0.conf — letzte Zahl durch 0 ersetzen
# Beispiel: Address = 10.8.0.5/24 → 10.8.0.0/24 verwenden
table <wg_internal>       persist { 10.8.0.0/24 }
```

WireGuard-Port anpassen — aus `[Peer] → Endpoint` in `wg-0.conf`:

```sh
# Diese Zeile in pf/pf.conf finden und Port aktualisieren:
pass out log quick on $wifi proto udp to <wg_endpoints> port { IHR_PORT } keep state
```

#### 4.2 — Regeln installieren

```bash
sudo cp pf/pf.conf /etc/pf.anchors/custom
```

#### 4.3 — Anchor zur Systemdatei hinzufügen

Nur zwei Zeilen werden hinzugefügt — nichts anderes ändert sich.

```bash
grep -n "custom" /etc/pf.conf   # prüfen ob bereits vorhanden
sudo cp /etc/pf.conf /etc/pf.conf.backup
sudo nano /etc/pf.conf
```

Vor `anchor "com.apple/*"` einfügen:

```sh
##### (5) FILTERING

# Eigene Regeln — laufen zuerst, vor Apple-Systemfiltern
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"

anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
```

Speichern: `Strg+O` → `Enter` → `Strg+X`

#### 4.4 — Syntax prüfen und laden

```bash
# Validieren — keine Ausgabe bedeutet keine Fehler
sudo pfctl -n -f /etc/pf.conf

# Laden und aktivieren
sudo pfctl -f /etc/pf.conf && sudo pfctl -e

# Aktive Regeln bestätigen
sudo pfctl -a custom -sr | head -20
```

### Schritt 5 — LaunchDaemon installieren

Lädt PF-Regeln bei jedem Boot. Enthält 5-Sekunden-Delay für Interface-Initialisierung. `WatchPaths` löst automatisches Reload bei `pf.conf`-Änderungen aus.

```bash
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist

sudo launchctl list | grep pfctl
cat /var/log/pf_autoload.log
```

### Schritt 6 — Verifizierung

```bash
sudo wg-quick up wg-0
bash ~/scripts/check-leaks.sh
```

Online-Überprüfung:
- https://dnsleaktest.com — Erweiterter Test
- https://ipleak.net
- https://browserleaks.com/webrtc

**Kill-Switch testen:**

```bash
curl https://ifconfig.me               # muss VPN-Server-IP zeigen
sudo wg-quick down wg-0               # trennen
curl --max-time 5 https://ifconfig.me  # muss Timeout ergeben
sudo wg-quick up wg-0                 # wieder verbinden
```

### Prüfskript

```bash
bash ~/scripts/check-leaks.sh
bash ~/scripts/check-leaks.sh --no-killswitch   # ohne Kill-Switch-Test
```

Bei korrekter Konfiguration:

```
  Passed    8
  Warnings  0
  Failed    0

  All checks passed.
```

### Fehlerbehebung

| Problem | Befehl |
|---------|--------|
| Kein Internet nach PF-Laden | `sudo pfctl -n -f /etc/pf.conf` — Syntaxfehler prüfen |
| Notfall — Zugang wiederherstellen | `sudo pfctl -d` — PF vollständig deaktivieren |
| DNS nicht aufgelöst | `brew services restart dnscrypt-proxy` + `dig google.com @127.0.0.1` |
| WireGuard verbindet nicht | `sudo wg show` + Endpoint IP/Port in pf.conf prüfen |
| PF-Regeln nach Neustart verloren | `cat /var/log/pf_autoload_error.log` |
| DNS-Resolver prüfen | `scutil --dns \| grep nameserver` — muss `127.0.0.1` zeigen |

### Nützliche Befehle

```bash
# WireGuard
sudo wg-quick up wg-0 / down wg-0      # verbinden / trennen
sudo wg show                            # Tunnel-Status und Peer-Statistiken

# PF
sudo pfctl -e / -d                      # aktivieren / deaktivieren
sudo pfctl -f /etc/pf.conf             # Regeln neu laden
sudo pfctl -n -f /etc/pf.conf         # Syntax prüfen ohne zu laden
sudo pfctl -a custom -sr               # geladene Custom-Anchor-Regeln anzeigen
sudo pfctl -ss | grep utun             # aktive States auf VPN-Interface
sudo pfctl -t abusers -T show          # aktuell rate-limitierte IPs
sudo pfctl -t abusers -T flush         # Abusers-Tabelle leeren

# DNSCrypt
brew services restart dnscrypt-proxy
dig google.com @127.0.0.1
dig +short TXT whoami.ds.akahelp.net   # prüfen welche IP der Resolver sieht

# Logs
tail -f /var/log/pf_autoload.log
tail -f /opt/homebrew/etc/dnscrypt-proxy.log
sudo tcpdump -n -i en0 port 53         # sollte leer sein — alles DNS auf Loopback
```


---

<a name="russian"></a>
#  Русский

## Почему я это сделал

Большинство VPN приложений работают нормально в обычных условиях — но два поведения меня беспокоили.

Первое: когда VPN соединение неожиданно обрывается, трафик без предупреждения переходит на обычную сеть. Второе: DNS запросы часто полностью обходят VPN туннель — даже при активном подключении.

Я хотел понять как на самом деле работает macOS PF файрвол и использовать его чтобы закрыть обе проблемы: kill-switch на уровне файрвола который останавливает трафик при обрыве туннеля, и зашифрованный DNS который остаётся на loopback независимо от состояния соединения.

Этот проект — результат проработки этого с нуля.

---

## Для кого это

- Системные администраторы и сетевые инженеры интересующиеся конфигурацией macOS файрвола
- Разработчики которые хотят понять как PF правила, DNS резолюция и VPN маршрутизация взаимодействуют на практике
- Пользователи которым нужно более предсказуемое VPN поведение чем дают большинство приложений
- Те кто предпочитает читать конфиг файлы а не доверять black-box приложениям

**Это не plug-and-play инструмент.** Требуется self-hosted WireGuard сервер, доступ к терминалу и готовность редактировать конфигурационные файлы.

---

## Практический контекст

Этот setup наиболее полезен когда:

- Вы регулярно используете сети которые не контролируете (Wi-Fi отеля, конференции, коворкинг) и хотите предсказуемое поведение
- Вы тестируете или изучаете поведение PF правил на macOS
- Вы хотите понять как DNS резолюция, VPN туннелирование и политика файрвола взаимодействуют на уровне ОС
- Вы хотите чтобы сеть вела себя одинаково каждый раз, без молчаливого fallback

Это также практический учебный проект — синтаксис PF правил, DNS flow и конфигурация LaunchDaemon задокументированы inline и доступны для редактирования.

---

## Как это работает

**1. DNS обрабатывается локально**
Все DNS запросы идут на `127.0.0.1:53`. DNSCrypt-proxy получает их, шифрует и пересылает через relay сервер к upstream резолверу. Relay видит IP но не содержимое запроса. Резолвер видит запрос но не IP.

**2. PF принудительно направляет через туннель**
Файрвол блокирует весь исходящий трафик на физических интерфейсах — кроме минимума для поднятия WireGuard: DHCP, bootstrap DNS и WireGuard handshake.

**3. WireGuard несёт всё остальное**
Как только туннель поднят, весь TCP, UDP и ICMP трафик идёт через зашифрованный WireGuard туннель к VPN серверу.

Если WireGuard падает — PF не имеет pass правила для физического интерфейса, трафик останавливается до восстановления туннеля. Молчаливого fallback нет.

---

## Что реально улучшается

- **Трафик останавливается при обрыве туннеля.** PF блокирует весь исходящий трафик на физических интерфейсах пока WireGuard не активен.
- **DNS запросы стабильно идут через локальный резолвер.** Весь DNS идёт через DNSCrypt-proxy на `127.0.0.1`. Файрвол это обеспечивает — приложения не могут переопределить.
- **Поведение сети инспектируемо.** Правила — текстовые файлы. `sudo pfctl -a custom -sr` в любой момент показывает что активно.
- **Правила загружаются автоматически при boot.** LaunchDaemon обрабатывает это с небольшой задержкой для инициализации интерфейсов. Изменения `pf.conf` вызывают автоматический reload через `WatchPaths`.

**Что не меняется:** оператор VPN сервера по-прежнему видит ваши traffic destinations. Browser fingerprinting и cookie tracking не затронуты.

---

## Компромиссы

| Решение | Эффект |
|---------|--------|
| Блокировка UDP/443 (QUIC) | HTTP/3 отключён — браузеры автоматически переходят на HTTP/2 |
| Отключение IPv6 глобально | Предотвращает обход туннеля; ломает IPv6-only сервисы |
| DNS только через loopback | Несколько миллисекунд дополнительной DNS задержки |
| Anonymized DNS relay | Один дополнительный сетевой хоп — чуть выше DNS задержка |
| AirDrop / mDNS заблокированы | Локальное сетевое обнаружение не работает пока активно |

---

## Что этот setup адресует

| Поведение | Как обрабатывается |
|-----------|-------------------|
| Трафик при обрыве VPN | PF блокирует все физические интерфейсы без активного туннеля |
| DNS виден провайдеру | DNS заблокирован на loopback, зашифрован через DNSCrypt |
| Резолвер видит IP клиента | Anonymized relay routing — резолвер и relay видят только часть |
| DNS hijacking | Только whitelist upstream конфигурация |
| Bogon IP маршрутизация | Полная RFC bogon таблица на всех интерфейсах |
| TCP сканирование портов | Невалидные комбинации TCP флагов заблокированы |
| QUIC обходит DNS контроль | UDP 443/853 заблокирован внутри туннеля |
| IPv6 обходит туннель | IPv6 заблокирован на всех non-loopback интерфейсах |

**Не в scope:**
- Скомпрометированный VPN сервер
- Вредоносное ПО с root доступом
- Browser fingerprinting или account-based отслеживание

---

## Используемые технологии

| Технология | Роль | Версия |
|-----------|------|--------|
| **PF (Packet Filter)** | Stateful файрвол, kill-switch, политика трафика | встроен в macOS |
| **WireGuard** | Протокол VPN туннеля | wireguard-tools + wireguard-go |
| **DNSCrypt-proxy** | Зашифрованный DNS с relay маршрутизацией | v2.1.15 |
| **launchd / LaunchDaemon** | Boot-персистентность, WatchPaths reload | встроен в macOS |
| **Homebrew** | Управление пакетами | 4.x |

---

## Quick Start

```bash
# 1. Установить зависимости
brew install wireguard-tools wireguard-go dnscrypt-proxy

# 2. Сгенерировать WireGuard ключи
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey \
          | wg pubkey > /opt/homebrew/etc/wireguard/publickey
# privatekey → вставить в PrivateKey в wg-0.conf
# publickey  → отправить администратору сервера
rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey

# 3. Создать конфигурацию WireGuard
# Важно: использовать wg-0.conf (с дефисом), не wg0.conf
sudo nano /opt/homebrew/etc/wireguard/wg-0.conf
# Заполнить: PrivateKey, Address, DNS=127.0.0.1, PublicKey, Endpoint, AllowedIPs=0.0.0.0/0
sudo chmod 600 /opt/homebrew/etc/wireguard/wg-0.conf

# 4. Настроить DNSCrypt
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
# Отредактировать: routes в [anonymized_dns], ключ: openssl rand -hex 32
brew services start dnscrypt-proxy

# 5. Установить системный DNS на 127.0.0.1
# Системные настройки → Сеть → Wi-Fi → Подробнее → DNS → 127.0.0.1

# 6. Заполнить таблицы pf/pf.conf, затем установить и загрузить
sudo cp pf/pf.conf /etc/pf.anchors/custom
sudo pfctl -n -f /etc/pf.conf   # сначала проверить
sudo pfctl -f /etc/pf.conf && sudo pfctl -e

# 7. Установить LaunchDaemon
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist

# 8. Подключить и проверить
sudo wg-quick up wg-0
bash ~/scripts/check-leaks.sh
```

---

## Полная инструкция по установке

### Предварительные требования

- macOS 26.4 или новее
- Работающий WireGuard сервер с: публичным ключом · IP и портом · VPN подсетью
- Терминал (Программы → Утилиты → Терминал)

### Шаг 1 — Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew --version
```

### Шаг 2 — Установить и настроить WireGuard

```bash
brew install wireguard-tools wireguard-go
```

Сгенерировать пару ключей:

```bash
sudo mkdir -p /opt/homebrew/etc/wireguard
wg genkey | tee /opt/homebrew/etc/wireguard/privatekey \
          | wg pubkey > /opt/homebrew/etc/wireguard/publickey

cat /opt/homebrew/etc/wireguard/privatekey   # → вставить в PrivateKey
cat /opt/homebrew/etc/wireguard/publickey    # → отправить администратору сервера

rm /opt/homebrew/etc/wireguard/privatekey /opt/homebrew/etc/wireguard/publickey
```

> Путь: Apple Silicon → `/opt/homebrew/etc/wireguard/` · Intel → `/usr/local/etc/wireguard/`
> Определить чип: `which wg-quick`

> **Важно:** Назовите конфиг `wg-0.conf` (с дефисом), не `wg0.conf`. Имя файла без дефиса может не запуститься с `wg-quick` на macOS с Homebrew.

```bash
sudo nano /opt/homebrew/etc/wireguard/wg-0.conf
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
sudo chmod 600 /opt/homebrew/etc/wireguard/wg-0.conf
sudo wg-quick up wg-0 && sudo wg show
```

### Шаг 3 — Установить DNSCrypt-proxy

```bash
brew install dnscrypt-proxy
cp dnscrypt/dnscrypt-proxy.toml /opt/homebrew/etc/dnscrypt-proxy.toml
```

Серверы: https://dnscrypt.info/public-servers — протокол `DNSCrypt`, `No logs` ✓, `DNSSEC` ✓

Relay: другая организация + другая страна чем у сервера.

```bash
nano /opt/homebrew/etc/dnscrypt-proxy.toml
```

```toml
# Секция [anonymized_dns]
routes = [
    { server_name='ВАШ_СЕРВЕР', via=['ВАШ_RELAY_1', 'ВАШ_RELAY_2'] },
]
```

**Сгенерировать ключ шифрования для лог-файлов:**

Этот ключ шифрует IP адреса в локальных DNSCrypt лог-файлах.
Сгенерируйте его и вставьте в `dnscrypt-proxy.toml`:

```bash
openssl rand -hex 32
```

Найдите этот раздел в `dnscrypt-proxy.toml` и вставьте вывод как значение ключа:

```toml
[ip_encryption]
  algorithm = "ipcrypt-ndx"
  key = "ВСТАВЬТЕ_СЮДА_64_СИМВОЛЬНЫЙ_HEX_КЛЮЧ"
```

> Держите этот ключ в тайне — не коммитьте его в публичный репозиторий.

Скопировать blocklists и запустить:

```bash
mkdir -p /opt/homebrew/etc/dnscrypt-proxy
cp dnscrypt/blocked-names.txt /opt/homebrew/etc/dnscrypt-proxy/
cp dnscrypt/blocked-ips.txt   /opt/homebrew/etc/dnscrypt-proxy/

brew services start dnscrypt-proxy
dig google.com @127.0.0.1
```

Системный DNS: **Системные настройки → Сеть → Wi-Fi → Подробнее → DNS** → `127.0.0.1`, удалить остальные.

### Шаг 4 — Настроить PF Firewall

> **Два отдельных файла:**
>
> `/etc/pf.conf` — системный файл macOS. Добавить только две строки (ссылка на anchor). IP адреса здесь не указывать.
>
> `/etc/pf.anchors/custom` — ваш набор правил. Здесь заполнять все таблицы и правила.

#### 4.1 — Отредактировать файл ваших правил

Открыть `pf/pf.conf` из этого репозитория и заполнить четыре таблицы:

```sh
# IP WireGuard сервера — из [Peer] → Endpoint в wg-0.conf
table <wg_endpoints>      persist { IP_ВАШЕГО_WG_СЕРВЕРА }

# Обычный DNS только до поднятия туннеля
# Как только туннель активен — весь DNS через DNSCrypt на 127.0.0.1
table <bootstrap_dns>     persist { 1.1.1.1 }

# IP вашего DNSCrypt upstream сервера
# Найти: dig +short ВАШ_DNSCRYPT_HOSTNAME
# При relay-only маршрутизации — оставить пустым
table <dnscrypt_upstream> persist { IP_ВАШЕГО_DNSCRYPT_СЕРВЕРА }

# Внутренняя подсеть WireGuard
# Из [Interface] → Address в wg-0.conf — заменить последнее число на 0
# Пример: Address = 10.8.0.5/24 → использовать 10.8.0.0/24
table <wg_internal>       persist { 10.8.0.0/24 }
```

Обновить WireGuard порт — найти в `[Peer] → Endpoint` в `wg-0.conf`:

```sh
# Найти эту строку в pf/pf.conf и обновить порт:
pass out log quick on $wifi proto udp to <wg_endpoints> port { ВАШ_ПОРТ } keep state
```

#### 4.2 — Установить правила

```bash
sudo cp pf/pf.conf /etc/pf.anchors/custom
```

#### 4.3 — Добавить anchor в системный pf.conf

Добавляются только две строки — ничего больше не меняется.

```bash
grep -n "custom" /etc/pf.conf   # проверить есть ли уже
sudo cp /etc/pf.conf /etc/pf.conf.backup
sudo nano /etc/pf.conf
```

Вставить **перед** `anchor "com.apple/*"`:

```sh
##### (5) FILTERING

# Ваши правила — выполняются первыми, до системных фильтров Apple
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"

anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
```

Сохранить: `Ctrl+O` → `Enter` → `Ctrl+X`

#### 4.4 — Проверить синтаксис и загрузить

```bash
# Проверить — нет вывода означает нет ошибок
sudo pfctl -n -f /etc/pf.conf

# Загрузить и включить
sudo pfctl -f /etc/pf.conf && sudo pfctl -e

# Убедиться что правила активны
sudo pfctl -a custom -sr | head -20
```

### Шаг 5 — Установить LaunchDaemon

Загружает PF правила при каждом boot. Включает задержку 5 секунд для инициализации интерфейсов. `WatchPaths` вызывает автоматический reload при изменениях `pf.conf`.

```bash
sudo cp launchdaemons/com.user.pfctl.enable.plist /Library/LaunchDaemons/
sudo chown root:wheel /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo chmod 644 /Library/LaunchDaemons/com.user.pfctl.enable.plist
sudo launchctl load -w /Library/LaunchDaemons/com.user.pfctl.enable.plist

sudo launchctl list | grep pfctl
cat /var/log/pf_autoload.log
```

### Шаг 6 — Верификация

```bash
sudo wg-quick up wg-0
bash ~/scripts/check-leaks.sh
```

Онлайн проверка:
- https://dnsleaktest.com — Extended test
- https://ipleak.net
- https://browserleaks.com/webrtc

**Тест kill-switch:**

```bash
curl https://ifconfig.me               # должен показать IP VPN сервера
sudo wg-quick down wg-0               # отключить
curl --max-time 5 https://ifconfig.me  # должен вернуть таймаут
sudo wg-quick up wg-0                 # подключить снова
```

### Скрипт проверки

```bash
bash ~/scripts/check-leaks.sh
bash ~/scripts/check-leaks.sh --no-killswitch   # без теста kill-switch
```

При корректной настройке в итоге:

```
  Passed    8
  Warnings  0
  Failed    0

  All checks passed.
```

> **Проверка 5 — DNS Resolver Identity:**
> Скрипт запрашивает `whoami.ds.akahelp.net` — диагностический домен который возвращает IP видимый upstream резолвером. При активном relay routing резолвер видит IP relay сервера, не ваш. IPv6 адрес relay (например `2c0f:...`) — ожидаем и корректен.

### Устранение неполадок

| Проблема | Команда |
|---------|---------|
| Нет интернета после загрузки PF | `sudo pfctl -n -f /etc/pf.conf` — проверить синтаксис |
| Аварийное восстановление | `sudo pfctl -d` — полностью отключить PF |
| DNS не резолвится | `brew services restart dnscrypt-proxy` + `dig google.com @127.0.0.1` |
| WireGuard не подключается | `sudo wg show` + проверить Endpoint IP/порт в таблицах pf.conf |
| Правила PF сброшены после перезагрузки | `cat /var/log/pf_autoload_error.log` |
| Проверить используемый DNS | `scutil --dns \| grep nameserver` — должен показать `127.0.0.1` |

### Полезные команды

```bash
# WireGuard
sudo wg-quick up wg-0 / down wg-0      # подключить / отключить
sudo wg show                            # статус туннеля и статистика peer

# PF
sudo pfctl -e / -d                      # включить / выключить
sudo pfctl -f /etc/pf.conf             # перезагрузить правила
sudo pfctl -n -f /etc/pf.conf         # проверить синтаксис без загрузки
sudo pfctl -a custom -sr               # показать активные правила custom anchor
sudo pfctl -ss | grep utun             # активные state'ы на VPN интерфейсе
sudo pfctl -t abusers -T show          # текущие rate-limited IP
sudo pfctl -t abusers -T flush         # очистить таблицу abusers

# DNSCrypt
brew services restart dnscrypt-proxy
dig google.com @127.0.0.1
dig +short TXT whoami.ds.akahelp.net   # проверить какой IP видит резолвер

# Логи
tail -f /var/log/pf_autoload.log
tail -f /opt/homebrew/etc/dnscrypt-proxy.log
sudo tcpdump -n -i en0 port 53         # должно быть пусто — весь DNS на loopback
```

---

## License

MIT — free to use, modify, and distribute.

## Contributing

Issues and pull requests are welcome. Please open an issue before submitting
a PR to discuss the proposed change.

---

*Tested on: macOS 26.4 · M1 · DNSCrypt-proxy v2.1.15 · wireguard-tools · wireguard-go*
