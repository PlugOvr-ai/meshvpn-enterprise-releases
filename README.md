# meshvpn enterprise

Binaries and installers of **meshvpn enterprise**: the open-source [meshvpn](https://github.com/PlugOvr-ai/meshvpn)
mesh VPN plus device booking and central access control, for test benches, labs and AI compute clusters.

- **Bookings**: people and AI agents book devices: one or several at once, exclusive or shared, or "any 4 free nodes
  with tag `a100`". They come in from outside the network with nothing but their SSH key:
  `ssh -J jump@gw.example.com alice@dev1.mesh`.
- **Personal accounts with sudo, for the time of a booking**: during a booking, the device has an account `alice`
  that only alice's key can log in to, and only through the gateway. When the booking ends, the account is locked,
  sudo is removed and alice's processes stop, including open sessions.
- **Central policy**: users, groups, which devices can be booked (by whom, for how long, with or without sudo),
  gateways. Admins sign the policy and every node receives it; devices and gateways enforce it themselves.
- **Portal**: one web interface and REST API for all your networks. Login is password plus two-factor code; AI
  agents use API keys and have the same rights as users.
- **Audit log**: who changed and booked what, and when.

Try it free for 30 days. A license unlocks the features afterwards (see [Licenses](#licenses) and [plugovr.ai](https://plugovr.ai)).

**Contents:** [Install](#install) · [Set up a network](#set-up-a-network) · [Bookings](#bookings-and-access) ·
[The portal](#the-portal) · [AI agents](#ai-agents) · [Licenses](#licenses) · [Updates](#updates) ·
[Files of a release](#files-of-a-release) · [Security](#security)

## Install

### Linux and macOS

Run on every node (devices, gateways, admin machines):

```sh
curl -fsSL https://github.com/PlugOvr-ai/meshvpn-enterprise-releases/releases/latest/download/install.sh | sudo sh
```

To install and join a network in one step (this also starts the service), use:

```sh
curl -fsSL https://github.com/PlugOvr-ai/meshvpn-enterprise-releases/releases/latest/download/install.sh | sudo sh -s -- join mesh1-...
```

- **Linux**: x86_64, ARM64 and ARMv7 (Raspberry Pi 2-5, Jetson, ARM servers). The binary is static, so it runs on
  any distribution. It runs as a systemd service.
- **macOS**: Apple Silicon and Intel. It runs as a launch daemon.

The installer checks the SHA-256 checksum of every download.

### Windows 10/11

In an **administrator** PowerShell (Windows PowerShell 5.1 or PowerShell 7):

```powershell
$env:MESHVPN_EDITION = "enterprise"; irm https://github.com/PlugOvr-ai/meshvpn/releases/latest/download/install.ps1 | iex
```

This is the open-source installer; `MESHVPN_EDITION` makes it fetch the enterprise build from this repository. It
installs to `C:\Program Files\meshvpn` with the [Wintun](https://www.wintun.net) driver and runs meshvpn as a task
that starts at boot. Windows machines can be admin and user machines; bookable devices and gateways run Linux (or
macOS for gateways).

If Microsoft Defender SmartScreen warns about the new program, choose **More info → Run anyway**. If Defender
quarantines it, restore it under **Windows Security → Virus & threat protection → Protection history**. We're
working on code signing.

### Which machine needs what

| Role | Runs | Platforms |
|---|---|---|
| **Device** (bench machine, GPU server): what people book | meshvpn + sshd | Linux |
| **Gateway**: where people and agents come in from outside | meshvpn + sshd, reachable from outside (port 22) | Linux, macOS |
| **Admin node**: changes the policy | meshvpn | Linux, macOS, Windows |
| **Portal** (optional): web interface and API for everything | meshvpn-portal | Linux with systemd |
| **People and agents** | an SSH client, nothing else | any |

## Set up a network

On the first machine, which becomes the network's admin:

```sh
sudo meshvpn init --network lab --endpoint gw.example.com:7870   # prints an invite
sudo meshvpn install                                             # service, also after reboots
sudo meshvpn license trial --email you@example.com --accept-terms  # 30 days, all features
```

On every other machine, with an invite from `sudo meshvpn invite`:

```sh
sudo meshvpn join mesh1-...        # (or the one-step installer above)
sudo meshvpn install
```

Then tag the machines and say who may book what. Run these on any node; changes need root on an admin node:

```sh
sudo meshvpn tag add bench                                  # on each bench device
sudo meshvpn tag add gateway                                # on the gateway

sudo meshvpn user add alice --key-file alice.pub            # people and agents, with their SSH public keys
sudo meshvpn user add agent7 --key-file agent7.pub --agent
sudo meshvpn group set team alice agent7
sudo meshvpn device add tag:bench --who group:team --max 8h # bookable by the team, up to 8 h, with sudo
sudo meshvpn gateway add tag:gateway
```

Each device then shows up in `meshvpn device list`, together with its bookings. For devices without sudo, add
`--no-sudo` to `device add`.

Open the gateway's SSH port (22) to wherever your users are. meshvpn itself uses port 7870 (TCP and UDP) on nodes
that others connect to.

## Bookings and access

```sh
sudo meshvpn book dev1 dev2 --user alice --for 2h          # exclusive: nobody else in that time
sudo meshvpn book dev3 --user bob --for 1d --shared        # shared with other shared bookings
sudo meshvpn book tag:bench --count 4 --user agent7 --from +1h --for 3h   # any 4 free devices
meshvpn bookings                                           # current and upcoming
sudo meshvpn cancel <id>
```

During the booking, alice logs in from anywhere. She needs only her key; she doesn't need meshvpn:

```sh
ssh -J jump@gw.example.com alice@dev1.mesh
```

Or put it in `~/.ssh/config` once:

```
Host *.mesh
    ProxyJump jump@gw.example.com
```

Then `ssh alice@dev1.mesh`, `scp`, `rsync` and VS Code Remote SSH work as usual. Outside her bookings, the gateway
doesn't let her key through.

## The portal

One web interface and REST API for all networks. It runs on a Linux server with systemd:

```sh
curl -fsSL https://github.com/PlugOvr-ai/meshvpn-enterprise-releases/releases/latest/download/install-portal.sh \
  | sudo sh -s -- --public-host portal.example.com --admin ops
```

- **Without a certificate**, the portal listens on `127.0.0.1:8080`. Put an HTTPS reverse proxy in front, e.g.
  Caddy: `portal.example.com { reverse_proxy 127.0.0.1:8080 }`.
- **With a certificate**, it serves HTTPS itself on port 443:
  `... --tls-cert /path/cert.pem --tls-key /path/key.pem`.
- **First login** sets up the second factor (any authenticator app).
- **Networks:** under **Networks**, create a network, or join an existing one with an invite. In the second case,
  an admin there runs `sudo meshvpn admin add portal`. Nodes join with invites from the portal (**Invite a node**).
  Each network uses ports 7900 and up (TCP and UDP) on the portal host, so open them in the firewall.
- **Licenses:** start a trial or install a license per network, after ticking "I accept the license terms".
- **Accounts:** users log in, book devices, and manage their SSH keys and API keys. Admins manage users, groups,
  device rules, gateways and licenses. If someone loses their phone, run `sudo meshvpn-portal reset2fa <user>`.

## AI agents

Agents are users with the `agent` flag. They book and connect like people, through the CLI on a node or through the
portal's API with an API key:

```sh
curl -H "Authorization: Bearer mvk_..." https://portal.example.com/api/networks
curl -H "Authorization: Bearer mvk_..." -H "Content-Type: application/json" \
     -d '{"network": "lab", "selector": "tag:a100", "count": 4, "duration": "3h"}' https://portal.example.com/api/book
```

`meshvpn guide` on any node prints a guide for agents: how to work with the network, and what's in it right now.

## Licenses

Start with a free **30-day trial** on an admin node or in the portal:
`sudo meshvpn license trial --email you@example.com --accept-terms`.

Plans and prices: **[plugovr.ai](https://plugovr.ai)**. Questions and licenses: sales@plugovr.ai.

```sh
sudo meshvpn license install meshvpn-L-....license --accept-terms
meshvpn license show
```

A license is installed once, on an admin node or in the portal; it reaches every node of the network. When a license
expires, the last policy stays in force, so devices stay protected, but it can't be changed until a new license is
installed.

Use of the enterprise features is subject to the license terms: <https://licenses.plugovr.ai/terms>
(`meshvpn license terms`). Starting a trial or installing a license means accepting them; the network's audit log
records who accepted which version, and when. (Releases up to v0.16.1 don't ask yet; there, leave out
`--accept-terms`.)

## Updates

```sh
sudo meshvpn update              # on each node: installs the latest enterprise release
```

Nodes check for new releases every few hours and show them in `meshvpn status`, but never install them by
themselves. To update the portal, run `install-portal.sh` again; it keeps its data in `/var/lib/meshvpn-portal`. To
pin a version, set `MESHVPN_VERSION=v0.16.1` for the installers.

## Files of a release

| File | |
|---|---|
| `install.sh` | installer for Linux and macOS nodes (the same as `install-enterprise.sh`) |
| `install-portal.sh` | installer for the portal (Linux, systemd) |
| `meshvpn-<target>.tar.gz` / `.zip` | the enterprise `meshvpn` program: Linux (`x86_64`, `aarch64`, `armv7` musl), macOS (`aarch64`/`x86_64-apple-darwin`), Windows (`x86_64-pc-windows-msvc`, with `wintun.dll`) |
| `meshvpn-portal-<target>.tar.gz` | the portal (Linux `x86_64`, `aarch64`) |
| `*.sha256`, `SHA256SUMS` | checksums |
| `meshvpn-enterprise-<version>.cdx.json` | software bill of materials (CycloneDX) |
| `EULA.md`, `SECURITY.md`, `README.txt` | license terms, security policy, short readme |

## Security

Please report vulnerabilities privately to **security@plugovr.ai**; see [SECURITY.md](SECURITY.md). Don't open public
issues for them.

---

meshvpn enterprise is built on the open-source [meshvpn](https://github.com/PlugOvr-ai/meshvpn) (Apache License 2.0).
Questions and licenses: sales@plugovr.ai. Technical contact: cw@plugovr.ai.
