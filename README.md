# Self-hosted HostAnywhere

Run the entire HostAnywhere control plane on your own server: your devices,
your mesh coordination, your backup keys, your dashboard — on hardware you
control. The hosted service at hostanywhere.io is unaffected by anything here.

**What's included:** device mesh, dashboard, encrypted phone backups,
storage servers, access control, device-local AI gateways, and public-URL
tunnels — every feature unlocked, no device or user caps.
**Not included:** billing (there's nothing to bill), enterprise SSO, and the
HostAnywhere-hosted AI gateway (device-local gateways work).

## Requirements

- A Linux host with Docker + Docker Compose, a public IP, and the WireGuard
  kernel module (any modern kernel has it).
- A domain you control, with these DNS records pointing at the host:
  - `A  <domain>` — dashboard + API
  - `A  *.<domain>` — public service URLs
  - `A  relay.<domain>` — relay fallback + tunnels
- Open ports: 80, 443 (TCP), 7443 (TCP), 51820 (UDP) — in your cloud
  provider's firewall **and** the OS firewall.

> **Cloudflare DNS users:** create all three records as **DNS only** (grey
> cloud, not Proxied — Cloudflare's default is Proxied). The server obtains
> its own TLS certificates and runs WireGuard, neither of which works behind
> the proxy.

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

## Social sign-in (optional)

Out of the box, sign-in is by email magic link (plus device tokens). The
Google / GitHub / Microsoft / Apple buttons light up automatically once you
register your **own** OAuth app with a provider and put its credentials in
`.env` — all callback URLs are built from your domain, so each provider just
needs to know it:

| Provider | Create the app at | Callback URL to register | `.env` vars |
|---|---|---|---|
| Google | [console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials → Create OAuth client ID (Web application) | `https://<domain>/auth/callback/google` | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` |
| GitHub | GitHub → Settings → Developer settings → OAuth Apps → New | `https://<domain>/auth/callback/github` | `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET` |
| Microsoft | [entra.microsoft.com](https://entra.microsoft.com) → App registrations → New (platform: Web; supported accounts: *any org directory + personal accounts*) | `https://<domain>/auth/callback/microsoft` | `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET` |
| Apple | [developer.apple.com](https://developer.apple.com) (paid program) → Services ID + Sign in with Apple key | `https://<domain>/auth/callback/apple` | `APPLE_SERVICE_ID`, `APPLE_TEAM_ID`, `APPLE_KEY_ID`, `APPLE_PRIVATE_KEY` |

Then `docker compose up -d` to apply. Notes:

- **Google** shows an "unverified app" interstitial until you complete their
  verification — fine for personal/team use (click Advanced → Continue).
- **Microsoft**: the client secret is under *Certificates & secrets* — copy
  the secret **Value**, not the ID.
- **Apple** requires the paid developer program and domain verification;
  most self-hosters skip it.

## Connecting devices

Download the desktop installers and the Android APK from this repo's
[Releases](https://github.com/hostanywhere/hostanywhere-selfhost/releases)
page (checksums in `SHA256SUMS.txt`); the iPhone app comes from the App
Store. They're the same apps hosted users run — you just point them at your
server:

- **Desktop (Mac/Windows/Linux):** three ways to point the agent at your
  server — pick one:
  1. **First-run page (easiest):** launch the tray app; the "Connect this
     device" page that opens has *Advanced: use a self-hosted server* — enter
     `https://<domain>` there, then sign in.
  2. **Windows fleet installs:**
     `HostAnywhere-setup-<ver>.exe /silent HACONTROL=https://<domain>`
     Sets the server machine-wide; every first-run on that machine then
     signs in against your server.
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

Everything is automatic — no wildcard certificate, DNS plugins, or custom
Caddy builds required:

- The apex and `relay.<domain>` get ordinary HTTP-01 certificates when the
  stack starts.
- Each exposed-service URL (`<name>.<domain>`) gets its own certificate on
  demand, the first time it's visited. The control plane approves issuance
  only for subdomains that actually exist in this deployment.
- Agent tunnel connections (`relay.<domain>:7443`) are TLS end-to-end: the
  relay serves the `relay.<domain>` certificate and picks up renewals
  automatically. The first tunnel may take a few extra seconds right after
  initial startup while the certificate is being issued.

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

## License

Free for personal use and internal business use — no device, user, or
feature limits. Offering HostAnywhere to others (managed hosting for
clients, resale, bundling) needs a commercial agreement — see
[LICENSE](LICENSE) or write to contact@hostanywhere.io. The
[hostanywhere-recover](https://github.com/hostanywhere/hostanywhere-recover)
tool is MIT-licensed so your exit path never depends on these terms.

## Support

Issues and questions: open a GitHub issue on this repo, or use the contact
options at hostanywhere.io/about — mention "self-host". Expect
community-level support, not the hosted SLA.
