# Pi-hole on Tanix TX3 (Amlogic S905X3) — Installation Guide

A clean, step-by-step procedure to install Pi-hole on a Tanix TX3 (or
similar Amlogic S905X3 TV-box) booting from USB with devmfc's Debian
Trixie image. Designed to work in a typical home network without
breaking access to the device during the procedure.

---

## Configuration variables

Replace these with your own values throughout the procedure. The defaults
below are reasonable for a Belgian/Dutch home network on a Netgear-style
router with the standard `192.168.1.0/24` LAN. Write down your chosen
values before starting — you'll need them in multiple steps.

| Variable | Description | Example |
|---|---|---|
| `<PIHOLE_IP>` | Static IP for the Pi-hole box, **outside** the DHCP range | `192.168.1.10` |
| `<PIHOLE_HOSTNAME>` | Hostname for the device (lowercase, no spaces) | `tanix-pihole` |
| `<ROUTER_IP>` | Your router's LAN IP | `192.168.1.1` |
| `<LAN_CIDR>` | Your LAN in CIDR notation | `192.168.1.0/24` |
| `<DHCP_RANGE_START>` | Start of new DHCP pool, leaving room for static IPs | `192.168.1.50` |
| `<DHCP_RANGE_END>` | End of DHCP pool | `192.168.1.250` |
| `<UPSTREAM_DNS_PRIMARY>` | Primary upstream DNS resolver | `9.9.9.9` (Quad9) |
| `<UPSTREAM_DNS_SECONDARY>` | Secondary upstream | `1.1.1.1` (Cloudflare) |
| `<TIMEZONE>` | Your timezone | `Europe/Brussels` |

---

## Hardware

- **Tanix TX3 (or similar Amlogic S905X3 TV-box)** — verify your board
  has gigabit ethernet (most do; check the PHY chip near the RJ45 port
  if uncertain, or check `dmesg | grep eth0` after first boot)
- **USB stick, minimum 8 GB** (16 GB recommended). USB 3.0 ideal for
  flash speed, though the box itself is mostly USB 2.0
- **Ethernet cable** — Pi-hole on Wi-Fi is a bad idea
- **Toothpick or cotton swab** — for the recessed reset button in the
  AV port (only needed if your box hasn't run `aml-multiboot-setup.sh`
  before, see [Boot procedure](#5-boot-procedure))

## Software prerequisites (on your PC)

- **7-Zip** — https://www.7-zip.org/ (to extract `.img.xz` files)
- **Rufus** — https://rufus.ie/ (to flash the image)
- **An SSH client** — Windows PowerShell, PuTTY, or similar
- **A good text editor** — Notepad++ on Windows, NOT plain Notepad (it
  mangles Unix line endings)

---

## Procedure overview

1. [Download and prepare the image](#1-download-and-prepare-the-image)
2. [Flash the USB stick](#2-flash-the-usb-stick)
3. [Configure boot.config for your box](#3-configure-bootconfig-for-your-box)
4. [Prepare router DHCP](#4-prepare-router-dhcp-range-and-reservation)
5. [Boot procedure on the Tanix](#5-boot-procedure)
6. [Initial system configuration](#6-initial-system-configuration)
7. [System updates](#7-system-updates)
8. [Install Pi-hole](#8-install-pi-hole)
9. [Tune Pi-hole](#9-tune-pi-hole)
10. [Verify and roll out](#10-verify-and-roll-out)

Optional appendices follow at the end:
- [A. Recommended security hardening (SSH keys, etc.)](#appendix-a-recommended-security-hardening)
- [B. Image-specific issues and recovery](#appendix-b--image-specific-issues-and-recovery)
- [C. log2ram, journald limits, aml-multiboot — when and how](#appendix-c-log2ram-journald-limits-aml-multiboot)
- [D. Troubleshooting](#appendix-d-troubleshooting)

---

## 1. Download and prepare the image

### 1.1 Pick the right image

Go to https://github.com/devmfc/debian-on-amlogic/releases.

Select an image release that matches **all** of:

- **`s905x3`** in the asset filename (not `s905x3-b`)
- **`bookworm`** or **`trixie`** for Debian (this guide assumes **Trixie**)
- **`Minimal`** in the name (no desktop environment)
- A kernel version on a **stable LTS line** — at the time of writing,
  6.12.x is the recommended sweet spot. Newer 6.18.x or 6.20.x kernels
  exist but are less battle-tested for 24/7 server use

Example asset name (used in this guide):
```
Devmfc_Debian-Trixie_6.12.56-meson64_Minimal-25.10.29.img.xz
```

Download the `.img.xz` file (about 100-150 MB compressed; 1-1.5 GB
when extracted).

> **Why not the latest kernel version?** For a system in the critical
> path of your network (DNS), pick boring over bleeding edge. A 6.12.x
> LTS kernel will get security backports for years; a 6.20.x mainline
> kernel may get replaced by 6.21 next month with whatever new bugs
> that brings.

### 1.2 Extract the image

Right-click the `.img.xz` file → **7-Zip** → **Extract here**. You'll
get a `.img` file of about 1-2 GB.

---

## 2. Flash the USB stick

> ⚠️ **All data on the stick will be erased.** Double-check you've
> picked the right drive letter in Rufus.

1. Insert your USB stick
2. Open **Rufus**
3. **Device** → select your USB stick
4. **Boot selection** → click **SELECT** → choose your `.img` file
5. **Partition scheme** → leave on what Rufus suggests (usually MBR)
6. **File system** → leave default
7. Click **START**
8. When Rufus asks **DD Image mode** vs **ISO Image mode** → choose
   **DD Image mode** (essential — ISO mode breaks this image)
9. Wait for completion (5-15 min)

After flashing, Windows may pop up "you need to format this disk"
warnings about the Linux ext4 partition. **Click cancel — do not format.**
That partition is fine; Windows just can't read ext4.

---

## 3. Configure boot.config for your box

This step is critical — without it, the image won't boot on your specific
box.

After flashing, Windows shows the FAT partition (~256 MB) of the stick
in Explorer. Open it.

Find the file `boot.config`. Open it with **Notepad++** (not Notepad —
Windows Notepad can corrupt line endings).

You'll see many lines starting with `#box=...`. Each line corresponds to
a specific TV-box model. Find the one matching your hardware.

For the **Tanix TX3 with S905X3 and gigabit ethernet**, the correct
entry is:
```
box=tanixtx3
```

Find the line `#box=tanixtx3` and remove the `#` to activate it.

Other Tanix TX3 variants in the same `boot.config` (use only one):
```
#box=tanixtx3            ← gigabit ethernet (most common, this is the one you want)
#box=tanixtx3_100M       ← 100 Mbit ethernet variant
#box=tanixtx3miniplus    ← Mini Plus variant
```

> **Verifying ethernet speed**: To confirm your ethernet capability
> after first boot: `dmesg | grep eth0` will show the negotiated link
> speed. Most Tanix TX3 boxes have gigabit despite varying labels on
> packaging — your dmesg is the authoritative source.

Save the file. Eject the USB stick safely.

---

## 4. Prepare router DHCP range and reservation

You want a static IP for Pi-hole — *outside* your DHCP range — so DHCP
never accidentally hands `<PIHOLE_IP>` to another device. Most consumer
routers (including Netgear stock firmware) only support address
reservation **inside** the DHCP range, so we shrink the DHCP range to
free up space for static infrastructure IPs.

### 4.1 Recommended IP layout for `192.168.1.0/24`

```
.1                    Router
.2  - .9              Network infrastructure (APs, switches, future stuff)
.10 - .19             Critical services (Pi-hole here)
.20 - .29             Storage / NAS
.30 - .39             Personal devices with reserved IP
.40 - .49             Reserve / printers / overflow static
.50 - .250            DHCP pool (200 addresses, plenty for a household)
.251 - .254           Admin reserve
```

### 4.2 On the router

Log in to your router's admin interface. The exact menu names vary
(this guide uses Netgear stock firmware terminology):

1. Find the **LAN Setup** / **DHCP Server** page
2. Set **Starting IP Address** to `<DHCP_RANGE_START>` (e.g. `192.168.1.50`)
3. Set **Ending IP Address** to `<DHCP_RANGE_END>` (e.g. `192.168.1.250`)
4. **Apply**

Don't set up the address reservation yet — we don't have the box's MAC
address until after first boot.

---

## 5. Boot procedure

There are two boot scenarios depending on whether your box has
"multiboot" installed in its bootloader.

### 5.1 First-time boot (no multiboot)

For a fresh Tanix TX3 that has **never** had `aml-multiboot-setup.sh`
run on it, the bootloader doesn't know to look for USB. You need to use
the reset-button trick:

1. Make sure the box is **fully unplugged** from power (not just off)
2. Insert the USB stick into a USB port
3. Locate the recessed reset button (in the AV port — push with a
   toothpick)
4. **Hold the reset button down** while you plug in power
5. **Keep holding** for about 7-10 seconds, then release
6. Wait — first boot takes longer because:
   - The root filesystem is being resized to fill the stick
   - SSH host keys are regenerated

### 5.2 Subsequent boots / box with multiboot installed

If you've previously run `./aml-multiboot-setup.sh` on this box, the
bootloader is already configured to look for USB on every boot. You
can simply:

1. Insert the USB stick
2. Plug in power

The box will boot from USB automatically. No reset button needed.

> **Should you install multiboot?** Convenience vs. risk tradeoff. The
> reset-button approach is bulletproof — if anything goes wrong with
> the USB image, just remove the stick and the box boots Android.
> Multiboot makes everyday reboots easier but slightly increases brick
> risk (very small in practice). For a Pi-hole that needs to come back
> up after a power outage without you being there to hold a button: yes,
> install multiboot. See [Appendix C](#appendix-c-log2ram-journald-limits-aml-multiboot).

### 5.3 Find the IP address after first boot

Wait 2-3 minutes after powering up. Then check your router's "attached
devices" page. The new device appears with a hostname like `tvbox`.
Note its current IP — that's where you SSH to.

```bash
ssh root@<current-IP>
```

> **Reinstalling on the same IP?** If you've used this IP for a Pi-hole
> install before (or any other Linux box), your SSH client has the old
> host key cached and will refuse to connect with a "REMOTE HOST
> IDENTIFICATION HAS CHANGED" warning. Clear the old key first:
> ```
> ssh-keygen -R <current-IP>
> ```
> (On Windows PowerShell or any OpenSSH client.) Then SSH again and
> accept the new fingerprint.

> **Alternative method — front display**: If your TV-box has a small
> front LED/VFD display (most Tanix TX3 models do), the devmfc image
> cycles through three pieces of information:
> - The current time in the system's defined timezone
> - The SoC temperature (typically around 45°C idle)
> - The **last octet of the IPv4 address** (e.g. `10` if your IP is
>   `192.168.1.10`)
>
> If you know your router's subnet (almost always `192.168.1.x` or
> `192.168.0.x`), the last octet alone is enough to construct the
> full IP. Useful when you don't have admin access to the router.

Default credentials:
- Username: `root`
- Password: `tvbox`

You should see the devmfc banner with system info (uptime, IP, kernel).
If yes — great, you have a working system. Continue.

> **Note on MAC addresses**: Amlogic boxes often use locally-administered
> MAC addresses (first octet typically `06:`, `0a:`, etc., not a real
> manufacturer prefix). Some boxes generate a new MAC on every boot from
> a hardware seed; others are stable. After the next step, verify your
> MAC remains stable across reboots.

---

## 6. Initial system configuration

You're now in a fresh Debian Trixie install booted from USB. First
order of business: housekeeping and a static IP.

### 6.1 Change the root password

```bash
passwd
```

Type a strong password.

### 6.2 Set hostname

```bash
hostnamectl set-hostname <PIHOLE_HOSTNAME>
```

Update `/etc/hosts` to match. Open it:

```bash
cat /etc/hosts
```

Find the line that maps the old hostname (`tvbox`) to `127.0.0.1`. The
devmfc image uses `127.0.0.1 tvbox` (single line, both `localhost` and
hostname on `127.0.0.1`). Replace `tvbox` with your new hostname:

```bash
sed -i 's/127.0.0.1 tvbox/127.0.0.1 <PIHOLE_HOSTNAME>/' /etc/hosts
```

Verify:

```bash
cat /etc/hosts
hostname
```

The prompt won't update until your next SSH session.

### 6.3 Set timezone

First check the current setting:

```bash
timedatectl
```

The devmfc image typically ships with a European timezone preset
(often `Europe/Amsterdam` or similar). If the displayed timezone
already matches your location, no action needed — skip to 6.4.

Only if the timezone is wrong:

```bash
timedatectl set-timezone <TIMEZONE>
```

> **NTP and timezone**: NTP only synchronizes the absolute time (UTC),
> not the timezone. Your timezone is set locally — by the image
> provider, by you, or by `tzdata` defaults. So if `timedatectl` shows
> the right time and timezone after first boot, that's because devmfc
> baked it into the image, not because NTP fixed it.

### 6.4 Find your MAC address

You'll need this for the DHCP reservation:

```bash
ip link show eth0
```

Look for the `link/ether xx:xx:xx:xx:xx:xx` line. Note the MAC.

### 6.5 Set up DHCP reservation on the router

Back to the router admin page:

1. Go to **LAN Setup → Address Reservation**
2. Add a new reservation:
   - **IP Address**: `<PIHOLE_IP>` (e.g. `192.168.1.10`)
   - **MAC Address**: the MAC you just noted
   - **Device Name**: `<PIHOLE_HOSTNAME>` or similar
3. **Apply**

### 6.6 Trigger the new lease

On the Tanix:

```bash
networkctl renew eth0
```

(devmfc images use `systemd-networkd`, hence `networkctl`. If that
doesn't work, fall back to `dhclient -r eth0 && dhclient eth0`.)

After renewal, your existing SSH session may freeze (your IP just
changed). Close it, and SSH to the new address:

```bash
ssh root@<PIHOLE_IP>
```

Verify with `ip addr show eth0` — you should see `inet <PIHOLE_IP>/24`.

### 6.7 Verify network and time sync

```bash
ping -c 3 1.1.1.1                # Layer 3 / IP routing
ping -c 3 deb.debian.org         # DNS resolution
timedatectl status               # Should show "synchronized: yes"
```

If any fail, troubleshoot before continuing — the next steps need
working internet and accurate time (HTTPS cert validation depends on
clock).

---

## 7. System updates

### 7.1 Run apt update and upgrade

```bash
apt update
apt upgrade -y
```

One prompt you may encounter during upgrade:

**`/etc/ssh/sshd_config` modified prompt**

devmfc has slightly modified this file (mainly to allow root login with
password). Choose **"keep the local version currently installed"** (`N`)
to retain SSH access via the default credentials. Otherwise you may lock
yourself out.

### 7.2 Install useful tools and pre-empt the logrotate prompt

```bash
apt install -y curl wget ca-certificates dnsutils net-tools htop
```

> **Expected output**: `dnsutils` will be silently substituted with
> `bind9-dnsutils` (Debian Trixie's replacement). Several packages
> (`ca-certificates`, `bind9-dnsutils`, `net-tools`) are likely already
> installed and will be skipped. The actual new installs are typically
> `curl`, `htop`, `wget` plus their dependencies.

**Critical step — install logrotate with the package-default config**:

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y \
  -o Dpkg::Options::="--force-confnew" logrotate
```

> **Why this step matters**: The devmfc image ships with a non-functional
> `/etc/logrotate.conf` (just `minsize 10M / maxsize 20M`, missing the
> standard `weekly`, `rotate 4`, and `include /etc/logrotate.d`
> directives). When `logrotate` is later pulled in as a dependency by
> Pi-hole's installer, dpkg detects the modified config and stops to
> ask whether to keep the local version or install the package
> maintainer's version. Pi-hole's installer redirects all output to
> `/dev/null`, so this prompt is **invisible** — the installer hangs
> indefinitely with no error message.
>
> By installing `logrotate` explicitly here, with `--force-confnew` to
> automatically accept the package-default config, we remove the
> hidden prompt before Pi-hole's installer reaches it.

Verify logrotate now has the proper config:

```bash
cat /etc/logrotate.conf
```

Expected output should include `weekly`, `rotate 4`, `create`, and most
critically `include /etc/logrotate.d`. Without `include`, per-package
log rotation snippets — including Pi-hole's own `/etc/logrotate.d/pihole`
that's installed later — would never be processed.

---

## 8. Install Pi-hole

### 8.1 Run the Pi-hole installer

With logrotate properly configured (step 7.2), the Pi-hole installer
should now run cleanly on Trixie.

```bash
cd ~
curl -sSL https://install.pi-hole.net | bash
```

> **Historical note**: Earlier versions of this guide (v1-v3) included
> a workaround here using `equivs` to build a `dnsutils` dummy package,
> because Pi-hole's `pihole-meta.deb` historically depended on the
> obsolete `dnsutils` package which doesn't exist in Trixie. **This is
> no longer required.** Pi-hole [PR #6444](https://github.com/pi-hole/pi-hole/pull/6444)
> (October 2025) updated the dependency to `bind9-dnsutils|dnsutils`
> using apt's alternatives syntax, which apt resolves correctly on
> Trixie. See [Appendix B.2](#b2-the-pi-hole-dnsutils-dependency-on-trixie-historical)
> for the historical context.

### 8.2 Installer prompts

The installer is interactive. Recommended answers:

| Prompt | Answer | Why |
|---|---|---|
| Network interface | `eth0` | Wired only |
| Upstream DNS provider | **Custom** | Pick your own |
| Custom DNS servers | `<UPSTREAM_DNS_PRIMARY>,<UPSTREAM_DNS_SECONDARY>` | Comma-separated, no spaces |
| Block lists (StevenBlack) | Yes | Solid baseline |
| Enable query logging | Yes | Required for the dashboard |
| Privacy mode | **0 - Show everything** | You're the admin of your own home |
| IPv4 | Confirm `<PIHOLE_IP>` | Auto-detected |
| IPv6 | **Skip** | Adds complexity, no benefit for most home setups |
| Default gateway | Confirm `<ROUTER_IP>` | Auto-detected |

> **Pi-hole 6 note**: If you've used Pi-hole 5 before, you'll notice
> the installer **no longer asks** about installing `lighttpd` or
> `php-cgi`. Pi-hole 6 has the web admin interface built into FTL
> itself. No external web server needed.

At the end, the installer shows a randomly generated **web admin
password**. **Note it down immediately** — you can reset it later with
`pihole setpassword`, but easier to grab it now.

### 8.3 Verify the installation

```bash
pihole status                                      # All components active?
ss -tlnp | grep -E ':(53|80|443)\s'                # FTL listening on ports?
dig @127.0.0.1 google.com +short                   # Should return real IPs
dig @127.0.0.1 doubleclick.net +short              # Should return 0.0.0.0
```

> **If `doubleclick.net` returns real IPs instead of `0.0.0.0`**: this
> is common immediately after install. Pi-hole has built the gravity
> database on disk (you'll see `[i] Number of gravity domains: ~80000`
> in the installer output), but FTL may not have loaded it into its
> in-memory blocking tree yet. While `pihole status` reports
> "blocking enabled", queries are still being forwarded upstream
> instead of being blocked.
>
> **The fix**: force a gravity reload (typically takes 5-10 seconds
> on the Tanix):
>
> ```bash
> pihole -g
> dig @127.0.0.1 doubleclick.net +short    # Should now return 0.0.0.0
> ```
>
> `pihole -g` will report `Status: No changes detected` for the
> blocklist source — that's expected, because the database content
> doesn't need updating. What it does do is rebuild and swap the
> in-memory gravity tree, which is what was missing. A reboot
> achieves the same effect but takes longer.

Then open in your browser:
```
http://<PIHOLE_IP>/admin
```

You should see the Pi-hole dashboard.

> **Cosmetic warning**: Some commands like `pihole status` may print
> `local: FTL_PID_FILE: readonly variable`. This is a known Pi-hole 6
> bash-script quirk, harmless. Functionality is unaffected.

---

## 9. Tune Pi-hole

The defaults are good but a few tweaks make a real difference.

### 9.1 Add additional blocklists

In the web UI: **Lists → Add a new list**.

Recommended additions on top of the default StevenBlack:

| Name | URL |
|---|---|
| OISD Big | `https://big.oisd.nl/` |
| HaGeZi Pro | `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/hosts/pro.txt` |

Optionally, for malware/phishing protection:

| Name | URL |
|---|---|
| HaGeZi TIF | `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/hosts/tif.txt` |

After adding lists: **Tools → Update Gravity → Update**.

> **Avoid**: Anti-Microsoft/anti-telemetry lists (will break Windows
> Update, Microsoft Store, OneDrive, Teams) and "ultimate"/"plus plus"
> lists (so aggressive they break legitimate sites).

### 9.2 Switch to Expert mode in Settings

Pi-hole 6 hides advanced settings by default. Top right of the Settings
page → toggle **Basic** → **Expert**.

### 9.3 Configure conditional forwarding

Lets Pi-hole show device hostnames in the dashboard instead of just IPs.

**Settings → DNS → Conditional Forwarding**:

In the text area, add this single line:
```
true,<LAN_CIDR>,<ROUTER_IP>
```

Example for the standard home setup:
```
true,192.168.1.0/24,192.168.1.1
```

Save & Apply.

> **Reality check on stock router firmware**: Some consumer routers
> (notably Netgear stock firmware) don't actually answer reverse-DNS
> queries, even though they know hostnames internally. If your dashboard
> still shows IPs after enabling this, your router is the limitation —
> not Pi-hole. Workaround: manually add Pi-hole client labels for your
> important devices in **Clients → Add a new client** with a comment.
> Long-term: consider custom firmware (Voxel for Netgear, OpenWrt) or
> letting Pi-hole take over DHCP.

### 9.4 DNS advanced settings

**Settings → DNS → Advanced**:

| Setting | Value | Reason |
|---|---|---|
| Never forward non-FQDNs | **On** | Keeps typos like "router" from leaking to upstream DNS |
| Never forward reverse lookups for private IP ranges | **Off** | Must be off for conditional forwarding to work |
| DNSSEC | **Off** | More problems than benefits in practice; Quad9 already validates upstream |

### 9.5 Database retention (optional)

**Settings → All Settings → Database**:

- `MAXDBDAYS`: change to **30** if you want to be lean (default in
  Pi-hole 6 is 91 days)

The FTL database will grow over time with query history. 30 days is
enough for any practical analysis and keeps the database lean. The
default of 91 is also reasonable; only change if you specifically
want a smaller database.

### 9.6 Set web admin password

If you missed the installer's password output:

```bash
pihole setpassword
```

### 9.7 Blocklist auto-update schedule

Pi-hole 6 automatically updates the gravity database (the merged
blocklist file) on a weekly schedule via systemd timers. You don't
need to configure anything for this to work.

To verify:

```bash
systemctl list-timers --all | grep -i pihole
systemctl status pihole-updateGravity.timer
```

You should see `pihole-updateGravity.timer` in `active (waiting)`
state.

To customize the schedule (or trigger a manual update):
- **Web UI**: Settings → All settings → Misc → Gravity
- **Manual run from terminal**: `pihole -g`
- **From web UI**: Tools → Update Gravity → Update

> **When you add a new blocklist** in the UI, gravity must run at
> least once for the new list to take effect. The next scheduled run
> will pick it up, but if you want it active immediately, trigger a
> manual update.

---

## 10. Verify and roll out

### 10.1 Test on one device first

Pick one device (your laptop, your phone) and manually configure its DNS
to `<PIHOLE_IP>` instead of the router. Use the device for an hour.

Things to verify:
- Web browsing works normally
- App functionality (YouTube, banking apps, etc.) works
- The Pi-hole dashboard shows queries from your device
- Some queries are blocked (red entries in Query Log)

### 10.2 If everything works after a day, roll out

Two options for network-wide deployment:

**Option A — Manual per-device**: Configure DNS to `<PIHOLE_IP>` on
each device. More work, but you keep visibility per client and can
easily revert any single device.

**Option B — Router-wide**: Set the router to advertise `<PIHOLE_IP>`
as the network DNS server. On Netgear stock firmware: **ADVANCED →
Setup → Internet Setup → Use These DNS Servers → Primary: `<PIHOLE_IP>`**,
secondary blank. Reboots clients (or wait for DHCP renewal) to pick
up the change.

> **Caveat with Option B on stock Netgear**: All queries will appear
> to come from `<ROUTER_IP>` in the Pi-hole dashboard, because the
> router runs a DNS forwarder. You lose per-client visibility unless
> you configure clients manually.

### 10.3 Done

You now have:
- A working Pi-hole on a Tanix TX3, booted from USB
- Network-wide ad/tracker blocking (or per-device, depending on choice)
- A clean Debian Trixie base for future tinkering
- An always-on appliance using a few watts of power

---

## Appendix A — Recommended security hardening

These are not strictly required for a LAN-only Pi-hole, but they're
good practice and inexpensive.

### A.1 SSH key authentication

Replace password auth with key auth. From your **client** (Windows
PowerShell on the machine you SSH from):

```powershell
# Generate a keypair (skip if you already have one)
ssh-keygen -t ed25519 -C "you@yourmachine"

# Copy your public key to the Tanix
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@<PIHOLE_IP> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh"
```

> **You'll be prompted for the root password** (the one you set in
> step 6.1) when running the second command — this is the *last time*
> you type the password before key auth takes over.

Test: open a new SSH session — you should log in without typing a
password. **Don't close your existing session until you've confirmed
key auth works in a fresh one.**

### A.2 Disable password authentication for SSH

After you've verified key auth works, optionally lock down SSH further.
Edit `/etc/ssh/sshd_config`:

```bash
nano /etc/ssh/sshd_config
```

Change (or add):
```
PasswordAuthentication no
PermitRootLogin prohibit-password
```

Apply:
```bash
systemctl restart ssh
```

> ⚠️ **Test in a new SSH session before closing your current one.**
> A misconfigured `sshd_config` can lock you out — your only fallback
> is HDMI + USB keyboard on the Tanix, or removing the USB stick and
> re-flashing.

### A.3 Create a regular user (optional)

For a single-purpose appliance with one admin, this is pure hygiene —
no functional benefit. Skip if you don't care.

```bash
adduser <yourname>
usermod -aG sudo <yourname>
# Copy your SSH key to the new user too
cp -r ~/.ssh /home/<yourname>/.ssh
chown -R <yourname>:<yourname> /home/<yourname>/.ssh
```

Test SSH as the new user, then disable root login entirely in
`sshd_config`:
```
PermitRootLogin no
```

---

## Appendix B — Image-specific issues and recovery

Two image quirks documented in detail.

### B.1 The logrotate.conf problem and Pi-hole installer hang

**The actual cause** (confirmed by devmfc in
[discussion #236](https://github.com/devmfc/debian-on-amlogic/discussions/236)):
the minimal devmfc images **don't include `logrotate` at all**. The
zram-config install script, however, assumes logrotate is present and
appends some tweaks (`minsize 10M / maxsize 20M`) to
`/etc/logrotate.conf`. Since that file doesn't exist on a fresh
minimal image, the script effectively *creates* it — but only with
those two lines, missing all the Debian defaults (`weekly`,
`rotate 4`, `create`, `include /etc/logrotate.d`).

**The result is a non-functional logrotate config**, but the more
serious consequence is that **Pi-hole's installer hangs indefinitely**
on systems where logrotate is installed for the first time as a
dependency of `pihole-meta`:

1. Pi-hole's installer runs `apt-get -qq ... install /tmp/pihole-meta.deb`
   with all output redirected to `/dev/null`
2. `pihole-meta` depends on `logrotate`, so apt installs it
3. During logrotate's configure phase, dpkg detects the locally-modified
   `/etc/logrotate.conf` and stops to ask whether to keep it or use the
   package version
4. The dpkg prompt is rendered to a pseudo-terminal but no human is
   watching that pty (the installer redirected output away)
5. dpkg waits forever for input that will never come
6. The installer appears to hang at `[i] Installing Pi-hole dependency package...`
   with zero progress and no error

**The fix** (now in section 7.2 of the main procedure): explicitly
install `logrotate` with `--force-confnew` *before* running the
Pi-hole installer:

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y \
  -o Dpkg::Options::="--force-confnew" logrotate
```

This installs logrotate, automatically accepts the package-default
config (replacing the broken devmfc version), and leaves nothing for
Pi-hole's installer to prompt about.

**What you see in the diff**:

The devmfc-shipped file:
```
minsize 10M
maxsize 20M
```

The Debian package version:
```
weekly
rotate 4
create
#dateext
#compress
include /etc/logrotate.d
```

The critical line is `include /etc/logrotate.d`. Without it, *none* of
the per-package rotation snippets dropped there get processed —
including Pi-hole's own `/etc/logrotate.d/pihole` snippet that's
installed during the Pi-hole installer run.

**Recovering from a hung Pi-hole installer**

If you ran the installer without first fixing logrotate and it hangs
at "Installing Pi-hole dependency package":

```bash
# In a SECOND SSH session (don't kill the first one yet):

# Confirm dpkg is stuck waiting on a hidden conffile prompt
ps aux | grep -E "dpkg|apt-get" | grep -v grep
# You should see apt-get and a dpkg --configure --pending process

# Check that /etc/logrotate.conf still has the broken devmfc version
cat /etc/logrotate.conf

# Now go back to the FIRST session and press Ctrl+C to abort the installer

# Wait 30 seconds, then in the second session check no apt/dpkg processes remain
ps aux | grep -E "dpkg|apt-get" | grep -v grep
# If they linger, kill them:
# kill <pid>  (gentle first, kill -9 only if needed)

# Now use the readline frontend (instead of the silent dialog frontend)
# to complete dpkg's pending operations — you'll see the prompt this time
DEBIAN_FRONTEND=readline dpkg --configure -a

# When the prompt appears, answer Y (install package maintainer's version)

# Verify the system is consistent
apt-get install -f -y
cat /etc/logrotate.conf | head -10   # Should show Debian defaults now

# Re-run the Pi-hole installer
curl -sSL https://install.pi-hole.net | bash
```

**This is being addressed**: devmfc has confirmed the zram-config
install script will be fixed in a future image release. If you're
following this guide on a devmfc image newer than v6.12.56, the
broken `/etc/logrotate.conf` may not exist at all — in which case
no fix is needed, and the explicit logrotate install in step 7.2
becomes a no-op.

Tracking discussion:
https://github.com/devmfc/debian-on-amlogic/discussions/236

### B.2 The Pi-hole dnsutils dependency on Trixie (historical)

Earlier versions of this guide included a workaround for a Pi-hole
installer failure on Debian Trixie. **As of Pi-hole `pihole-meta`
v0.7+ (released February 2026), this workaround is no longer
necessary.** This appendix is kept for historical reference and to
help anyone running into the original symptom on an older Pi-hole
release.

**The original problem**: Pi-hole's installer built a meta-package
`pihole-meta.deb` declaring a dependency on `dnsutils`. In Bookworm
this was a transitional package pointing to `bind9-dnsutils`. In
Trixie the transitional package was removed entirely, so installs
failed with:

```
Error: Unable to install Pi-hole dependency package
```

**The fix in upstream Pi-hole**: [PR #6444](https://github.com/pi-hole/pi-hole/pull/6444)
(merged October 2025) updated the dependency to
`bind9-dnsutils|dnsutils`, using apt's alternatives syntax. apt's
resolver picks `bind9-dnsutils` (which exists in Trixie) and the
install completes cleanly.

You can verify the fix is in your Pi-hole's installer by running:

```bash
grep -n "bind9-dnsutils" /etc/.pihole/automated\ install/basic-install.sh
```

If you see a line containing `bind9-dnsutils|dnsutils` in the
`Depends:` declaration, the fix is present.

**Empirical confirmation** (May 2026): a clean install of Pi-hole on
devmfc's Debian Trixie image (v6.12.56) succeeded without any
dnsutils workaround once the logrotate-prompt issue from
[Appendix B.1](#b1-the-logrotateconf-problem-and-pi-hole-installer-hang)
was addressed.

**Tracking issue**: https://github.com/pi-hole/pi-hole/issues/6436

**Legacy equivs-shim workaround** (only if you have an older Pi-hole
version that still hits this issue):

```bash
apt install -y equivs

mkdir -p /tmp/dnsutils-shim && cd /tmp/dnsutils-shim

cat > dnsutils-control <<EOF
Section: misc
Priority: optional
Standards-Version: 3.9.2
Package: dnsutils
Version: 1:9.20.21
Depends: bind9-dnsutils
Description: Transitional dummy package for dnsutils
 This is a dummy package that depends on bind9-dnsutils.
EOF

equivs-build dnsutils-control
apt install -y ./dnsutils_*.deb
```

### B.3 Bonus: SSH config diff (cosmetic only)

devmfc's `/etc/ssh/sshd_config` has two minor differences from the
Debian package:
1. A typo in a comment line (open quote not closed)
2. `Banner /etc/issue.net` enabled (shows the devmfc TVBOX banner on
   login)

Neither is functional. The actual `PermitRootLogin yes` is still in
the main sshd_config (not in `/etc/ssh/sshd_config.d/`). Choose "keep
local version" during apt upgrade to preserve the banner.

### B.4 Verifying logrotate is working

After the install, verify logrotate is properly scheduled:

```bash
# Timer should be active (waiting)
systemctl status logrotate.timer

# Service is "static" (oneshot, only runs when triggered) — this is correct
systemctl status logrotate.service

# When will logrotate next run?
systemctl list-timers logrotate.timer

# Dry-run to see what logrotate would do right now
logrotate -d /etc/logrotate.conf 2>&1 | head -50
```

Expected:
- `logrotate.timer` — `active (waiting)` with a "Trigger" time tomorrow morning
- `logrotate.service` — `inactive (dead)` between runs (it's a oneshot, not a daemon)

The message `logrotate.service is a disabled or a static unit, not
starting it` that you may have seen during install is **not** a
warning — it's dpkg confirming it didn't start the service directly,
because the timer manages that.

---

## Appendix C — log2ram, journald limits, aml-multiboot

Optional optimizations; do these *after* the base system is stable.

### C.1 Should you install log2ram?

**Probably not.** A typical Pi-hole on USB writes ~60 KB/day to
`/var/log` after journald limits are applied. A decent USB stick has
hundreds of TB of write endurance. The stick will die of physical
aging long before write wear.

Additionally, the devmfc image includes `zram-config` (with `/var/log`
on compressed zram in its `/etc/ztab`), but it's gated to **SD-card
boot only** via `/usr/lib/qubian/utils/if_rootfs_on_sdcard`. devmfc
explicitly chose not to enable log compression on USB — implicit
endorsement of "leave logs alone on USB".

If you do install log2ram (e.g., on an aggressively-used SD-card), see
the azlux repo at https://packages.azlux.fr/.

### C.2 Tighten journald limits (recommended, free)

Even without log2ram, cap journald to prevent runaway log growth:

```bash
nano /etc/systemd/journald.conf
```

Uncomment / set:
```
SystemMaxUse=64M
SystemMaxFileSize=16M
RuntimeMaxUse=64M
```

Apply:
```bash
systemctl restart systemd-journald
```

### C.3 Install aml-multiboot

Optional, removes the need to hold the reset button on every boot.
**Only do this once the system has been stable for a while** — and
only if you're prepared to deal with a small brick risk.

```bash
cd /root
./aml-multiboot-setup.sh
```

After running this:
- Future reboots from USB just work (no reset button needed)
- The bootloader environment in eMMC is modified — you cannot trivially
  uninstall this
- If something goes wrong, recovery requires Amlogic USB Burning Tool
  and a USB-A-to-A cable

For an always-on Pi-hole appliance that needs to come back up after
power outages without manual intervention: **highly recommended**.

---

## Appendix D — Troubleshooting

### D.1 Box doesn't boot from USB

- Verify `boot.config` is correctly edited (uncomment the right `box=` line)
- Verify Rufus used **DD Image mode**, not ISO mode
- Try a different USB port (some boxes are picky about which port)
- Try a different USB stick (cheap sticks sometimes have boot issues)
- If you've never run `aml-multiboot-setup.sh`: are you holding the
  reset button at the right time? It needs to be held *while* you
  apply power, for ~7-10 seconds.

### D.2 Lost SSH access after edit to sshd_config

If you can't log in over SSH and need to recover:

- Connect HDMI + USB keyboard to the Tanix → log in via console
- Or remove the USB stick → mount it on another Linux machine →
  fix `/etc/ssh/sshd_config` directly

### D.3 Pi-hole dashboard shows IPs instead of hostnames

Most likely your router doesn't answer reverse-DNS queries. Test with:
```bash
dig -x <some-client-IP> @<ROUTER_IP> +short
```
Empty result = router doesn't answer. Workarounds:
- Add Pi-hole client labels manually (UI: Clients → Add a new client)
- Switch to custom router firmware
- Let Pi-hole take over DHCP

### D.4 iOS/Android device not showing real hostname

iOS and Android default to "Private Wi-Fi Address" (random MAC per
network), which prevents DHCP from associating a stable hostname.

**iOS fix**: Settings → Wi-Fi → (i) next to your SSID → turn off
"Private Wi-Fi Address" → forget and reconnect.

**Android fix**: Wi-Fi settings → long-press SSID → Modify → Advanced →
Privacy → "Use device MAC".

### D.5 MAC address changes between reboots

Some Amlogic boxes don't have a fixed MAC stored in OTP/eFuse and
generate a new one each boot. Verify with:
```bash
ip link show eth0 | grep ether
reboot
# After reboot:
ip link show eth0 | grep ether
```

If MAC differs: pin it via `systemd-networkd` config. Edit
`/etc/systemd/network/10-eth0.network`:
```
[Link]
MACAddress=<your-chosen-mac>
```
Use the MAC that's currently in your DHCP reservation.

### D.6 Pi-hole installer hangs or fails

If the installer hangs silently at "Installing Pi-hole dependency
package" with no error or progress:

- Did you run the explicit logrotate install in step 7.2 with
  `--force-confnew`? This is the most common cause of silent hangs.
  See [Appendix B.1](#b1-the-logrotateconf-problem-and-pi-hole-installer-hang)
  for the recovery procedure.

If the installer fails with `Error: Unable to install Pi-hole
dependency package`:

- Is your Pi-hole release recent enough to have the dnsutils fix
  ([PR #6444](https://github.com/pi-hole/pi-hole/pull/6444), Oct 2025)?
  Check with: `grep "bind9-dnsutils" /etc/.pihole/automated\ install/basic-install.sh`
  If you don't see `bind9-dnsutils|dnsutils`, fall back to the legacy
  equivs-shim from [Appendix B.2](#b2-the-pi-hole-dnsutils-dependency-on-trixie-historical).
- Is `/etc/resolv.conf` pointing somewhere that can resolve names?
  Test: `dig deb.debian.org @1.1.1.1 +short`
- Is system time correct? `timedatectl status` — HTTPS cert validation
  fails if clock is off

### D.7 "Unsupported OS detected: Debian 13" when running `pihole -up`

Pi-hole as of writing doesn't officially mark Trixie as a supported
OS, even though installs work. `pihole -up` may refuse to update.
Workaround: re-run the installer to update Pi-hole:
```bash
curl -sSL https://install.pi-hole.net | bash
```

This effectively performs an in-place reinstall, picking up new versions.

---

## References

- devmfc/debian-on-amlogic — https://github.com/devmfc/debian-on-amlogic
- Pi-hole — https://pi-hole.net
- StevenBlack hosts — https://github.com/StevenBlack/hosts
- OISD blocklists — https://oisd.nl
- HaGeZi blocklists — https://github.com/hagezi/dns-blocklists
- Pi-hole issue #6436 (dnsutils dep) — https://github.com/pi-hole/pi-hole/issues/6436
- devmfc discussion #236 (logrotate) — https://github.com/devmfc/debian-on-amlogic/discussions/236

---

*Guide written based on hands-on installation experience on a Tanix
TX3-H (CS_905X3_TX95_B4_QZ_V1.2A board) running devmfc Debian Trixie
v6.12.56 (Minimal-25.10.29). Tested on a Belgian home network with
a Netgear R7800 router on stock firmware V1.0.3.92. Updated with
findings as of May 2026.*

*Version 2 — incorporates confirmed `box=tanixtx3` boot.config entry,
front-display IP discovery method, and devmfc's explanation of the
logrotate.conf issue from
[discussion #236](https://github.com/devmfc/debian-on-amlogic/discussions/236).*

*Version 3 — corrects placement of the logrotate prompt (appears
during dnsutils-shim install, not during apt upgrade), updates
MAXDBDAYS default (91 in Pi-hole 6, not 365), adds NTP-vs-timezone
clarification, adds blocklist auto-update info, adds SSH host-key
clearing tip for reinstalls, and adds a logrotate verification
section. Reflects feedback from a clean second-time installation.*

*Version 4 — based on a third clean install with deliberate testing,
the procedure has been substantially simplified. The dnsutils
equivs-shim workaround has been removed from the main flow because
[Pi-hole PR #6444](https://github.com/pi-hole/pi-hole/pull/6444)
makes it unnecessary for current versions. The actual cause of
silent installer hangs on this image was identified as the
locally-modified `/etc/logrotate.conf` triggering an invisible dpkg
conffile prompt during `pihole-meta` configuration. Step 7.2 now
explicitly installs `logrotate` with `--force-confnew` to pre-empt
this prompt before the Pi-hole installer runs. Appendix B.1 documents
the recovery procedure if you hit the hang anyway, and Appendix B.2
preserves the dnsutils workaround as historical reference for those
on older Pi-hole versions.*
