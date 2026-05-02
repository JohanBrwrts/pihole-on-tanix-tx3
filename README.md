# Pi-hole on a Tanix TX3 TV-box

A complete, hands-on installation guide for running [Pi-hole](https://pi-hole.net)
on a **Tanix TX3** (or similar Amlogic S905X3 TV-box) using
[devmfc's Debian Trixie image](https://github.com/devmfc/debian-on-amlogic).

## Why this guide?

Pi-hole tutorials on the internet overwhelmingly assume you're using a
Raspberry Pi. But cheap S905X3 TV-boxes have everything you need —
quad-core ARM64 CPU, gigabit ethernet, 4 GB RAM — and they're often
already lying in a drawer collecting dust after being replaced by a
newer streaming device. Repurposing them as a Pi-hole appliance is a
nice way to keep them useful, and it frees up your Raspberry Pi for
other projects.

The catch: getting Linux running smoothly on these boxes is its own
adventure. This guide walks through the complete setup, including
several gotchas that aren't well-documented elsewhere:

- The right `boot.config` entry for the Tanix TX3
- A workaround for Pi-hole's installer failing on Debian Trixie
  ([upstream bug](https://github.com/pi-hole/pi-hole/issues/6436))
- The misleading `/etc/logrotate.conf` modified prompt during install
  ([explained in upstream discussion](https://github.com/devmfc/debian-on-amlogic/discussions/236))
- Reverse-DNS limitations on stock Netgear router firmware
- iOS/Android private MAC quirks affecting Pi-hole's client visibility

## What's in this repo

- **[Installation guide](tanix-pihole-installation-guide.md)** — the
  complete, step-by-step procedure (~1000 lines). Start here.

The guide includes:
- Configuration variables to adapt to your own network
- Hardware and software prerequisites
- Image flashing and boot configuration
- Router DHCP setup with reservation outside the DHCP range
- Initial system configuration (hostname, timezone, network)
- System updates with handling of common dpkg prompts
- Pi-hole installation including the dnsutils workaround
- Pi-hole tuning (blocklists, conditional forwarding, DNS settings)
- Verification and rollout to network clients
- Optional appendices: SSH key auth, log management, troubleshooting

## Who is this for?

- You have an unused Amlogic S905X3 TV-box (Tanix TX3 specifically
  tested; similar boxes likely work with adjusted `boot.config`)
- You want network-wide ad and tracker blocking
- You're comfortable with SSH and a Linux terminal
- You want to understand *why* each step is there, not just blindly
  copy-paste commands

## Hardware tested

| Component | Detail |
|---|---|
| TV-box | Tanix TX3-H (S905X3, board `CS_905X3_TX95_B4_QZ_V1.2A`) |
| RAM/Storage | 4 GB / 64 GB eMMC (eMMC unused; boot from USB) |
| Ethernet | Gigabit (RTL8211F PHY) |
| OS image | devmfc Debian Trixie, kernel 6.12.56-meson64 |
| Boot medium | USB stick (~8 GB minimum, 16 GB recommended) |
| Router | Netgear R7800 stock firmware V1.0.3.92 |

The procedure should work on any Tanix TX3 variant and likely on
similar S905X3 boxes (X96, TX95) with the appropriate `boot.config`
entry. Other boxes may work with minor adjustments — the
[devmfc README](https://github.com/devmfc/debian-on-amlogic) has the
full compatibility list.

## Status

The procedure has been verified by performing a clean install from
scratch (a second-time install, not the original installation that
inspired it). Feedback from that second install is incorporated.

The Pi-hole `dnsutils` issue ([#6436](https://github.com/pi-hole/pi-hole/issues/6436))
is still open at the time of writing. The logrotate cause has been
[acknowledged by devmfc](https://github.com/devmfc/debian-on-amlogic/discussions/236)
and will be fixed in a future image release — at which point that
section of the guide can be skipped.

## Contributing

Found a step that doesn't work on your hardware? An updated package
name? A clearer way to explain something? Issues and pull requests
welcome. Please mention your hardware (TV-box model, image version)
and what you observed.

## License

See [LICENSE](LICENSE).

## Related projects

- [Pi-hole](https://pi-hole.net) — the network-wide ad blocker
- [devmfc/debian-on-amlogic](https://github.com/devmfc/debian-on-amlogic)
  — Debian and Ubuntu images for Amlogic TV-boxes
- [StevenBlack/hosts](https://github.com/StevenBlack/hosts) — the
  default Pi-hole blocklist
- [hagezi/dns-blocklists](https://github.com/hagezi/dns-blocklists) —
  recommended additional blocklists
- [oisd.nl](https://oisd.nl) — another excellent blocklist

## Acknowledgements

This guide builds on the work of:
- **devmfc** for maintaining `debian-on-amlogic`, which makes running
  modern Debian on Amlogic TV-boxes straightforward
- **The Pi-hole team** for the software itself
- **StevenBlack, OISD, HaGeZi** and other blocklist maintainers
- The various GitHub issue and forum threads that helped me piece
  together the dnsutils workaround
