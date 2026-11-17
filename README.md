# Self-hosted HostAnywhere (beta)

Run the entire HostAnywhere control plane on your own server: your devices,
your mesh coordination, your backup keys, your dashboard — on hardware you
control. The hosted service at hostanywhere.io is unaffected by anything here.

**Beta scope:** device mesh, dashboard, encrypted phone backups, storage
servers, access control, device-local AI gateways, public-URL tunnels.
**Not in the beta:** billing (everything is unlocked), SSO, and the
HostAnywhere-hosted AI gateway (device-local gateways work).

## Requirements

- A Linux host with Docker + Docker Compose, a public IP, and the WireGuard
  kernel module (any modern kernel has it).
- A domain you control, with these DNS records pointing at the host:
  - `A  <domain>` — dashboard + API
  - `A  *.<domain>` — public service URLs
  - `A  relay.<domain>` — relay fallback + tunnels
- Open ports: 80, 443 (TCP), 7443 (TCP), 51820 (UDP).

## Install

```sh
git clone https://github.com/hostanywhere/hostanywhere-selfhost && cd hostanywhere-selfhost
cp .env.example .env
# edit .env: set DOMAIN and JWT_SECRET (openssl rand -hex 32)
docker compose up -d
```

Images are prebuilt for amd64 and arm64
(`ghcr.io/hostanywhere/hostanywhere-control` and `…-relay`); nothing compiles
on your machine. Pin a version with `HA_VERSION=<tag>` in `.env` (defaults to
`latest`).

First sign-in: open `https://<domain>`, enter your email, then read the magic
link from the logs (no SMTP required):

```sh
docker compose logs -f control | grep -A2 "MAGIC LINK"
```

The first account you create is yours — in self-host mode every feature is
unlocked and there are no device or user caps.

## Connecting devices

- **Desktop (Mac/Windows/Linux):** three ways to point the agent at your
  server — pick one:
  1. **First-run page (easiest):** launch the tray app; the "Connect this
     device" page that opens has *Advanced: use a self-hosted server* — enter
     `https://<domain>` there, then sign in.
  2. **Windows fleet installs:**
     `msiexec /i HostAnywhere-setup-<ver>.msi HACONTROL=https://<domain>`
     (also accepted by the .exe installer: `HostAnywhere-setup.exe /silent
     HACONTROL=https://<domain>`). Sets the server machine-wide; every
     first-run on that machine then signs in against your server.
  3. **Env var:** `HA_CONTROL=https://<domain>` in the tray's environment, or
     `--control https://<domain>` for the CLI agent.

  After the first sign-in the value persists in the agent's config.
- **iPhone / Android:** on the sign-in screen tap **Advanced** and enter
  `https://<domain>`, then sign in as normal.

## Back up the control database

`ha_data` (the Docker volume) holds `hostanywhere.db` — device registry,
accounts, **and the escrowed backup encryption keys**. If you run phone
backups, losing this database and your phones in the same incident means
losing the keys — so back the volume up, and have each phone user export
their recovery code (Backups → Settings → Export recovery code). With a
recovery code and the files on a storage server, the open-source
[hostanywhere-recover](https://github.com/hostanywhere/hostanywhere-recover) tool restores everything with no server at
all.

```sh
docker compose exec control sh -c 'sqlite3 /data/hostanywhere.db ".backup /data/backup.db"'
docker cp "$(docker compose ps -q control)":/data/backup.db ./hostanywhere-$(date +%F).db
```

## TLS notes

- The apex certificate is automatic (Caddy, HTTP-01).
- The `*.<domain>` wildcard needs a DNS-challenge certificate: either build
  Caddy with your DNS provider's module and enable the `tls { dns ... }`
  line in the Caddyfile, or front this host with a proxy that brings its own
  wildcard. If you skip the wildcard, the dashboard and mesh work fully —
  only public service URLs on subdomains need it.
- Agent tunnel connections use `relay.<domain>:7443` (TLS passthrough isn't
  proxied by Caddy). Set `relay_addr` accordingly in the desktop agent's
  config when exposing services.

## Networking gotchas (read before filing an issue)

- **Provider firewalls sit in front of the OS.** Hetzner, OCI, AWS, Azure
  etc. all default-deny inbound: open 80 + 443 + 7443 (TCP) and 51820 (UDP)
  in the provider's firewall/security-group UI too, not just ufw/iptables —
  a container that binds the port still looks dead from outside otherwise.
- **Windows blocks inbound ICMP by default.** Pinging a Windows device's
  mesh IP times out even when the mesh is healthy. Either test with
  something TCP (`curl`, RDP, file share) or allow echo on that machine:
  `New-NetFirewallRule -DisplayName "ICMPv4 echo" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow`
- **Ping between two devices proves the mesh; ping *to the hub* may not** —
  the hub forwards peer-to-peer traffic and devices often hole-punch into
  a direct path (check the dashboard's peer view).

## Upgrades

```sh
git pull                    # picks up compose/Caddyfile changes
docker compose pull && docker compose up -d
```

The database migrates itself on boot. Take a DB backup first (see above).

## Support

This is a beta. Issues and questions: the contact options at
hostanywhere.io/about — mention "self-host beta". Expect community-level
support, not the hosted SLA.
