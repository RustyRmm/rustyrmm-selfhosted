# RustyRMM self-hosted

Run your own [RustyRMM](https://rustyrmm.com) server with Docker Compose: the
RustyRMM server and web console, a Postgres database, HTTPS through either
Cloudflare Tunnel or Let's Encrypt, and an optional RustDesk server for remote
desktop.

## Requirements

- A Linux host with Docker Engine 24+ and Docker Compose v2. Ubuntu 24.04 is
  a good choice.
- At least 2 vCPU, 2 GB RAM and 20 GB disk.
- A domain name for the console, for example `rmm.example.com`.

## 1. Choose how the console is reached

| | Cloudflare Tunnel (default) | Let's Encrypt (Caddy) |
|---|---|---|
| Inbound ports for the console | None | 80 and 443 |
| Public IP / port forwarding | Not needed | Needed |
| DNS | Domain on Cloudflare | Any provider: an `A` record pointing at this host |

**Cloudflare Tunnel:** in the Cloudflare dashboard go to **Zero Trust →
Networks → Tunnels**, create a tunnel and copy its token. Add a public
hostname (for example `rmm.example.com`) with the service
`http://rustyrmm:8080`.

**Let's Encrypt:** point the domain's DNS record at this host before the
first start, and open ports 80 and 443.

**Remote desktop (optional):** the `rustdesk` profile runs a RustDesk server
next to RustyRMM. It needs TCP 21115–21117 and UDP 21116 open to this host.
With Cloudflare Tunnel, give it its own DNS-only (grey cloud) record, for
example `rustdesk.example.com`, because tunnelled names don't carry those
ports.

## 2. Get the files and configure

```sh
git clone https://github.com/RustyRmm/rustyrmm-selfhosted.git
cd rustyrmm-selfhosted
cp sample.env .env
```

Edit `.env`:

| Setting | What to put |
|---|---|
| `COMPOSE_PROFILES` | `tunnel,rustdesk` (Cloudflare Tunnel), `caddy,rustdesk` (Let's Encrypt), or leave out `rustdesk` if you don't want remote desktop |
| `RMM_DOMAIN` | The console's name, e.g. `rmm.example.com` |
| `CLOUDFLARE_TUNNEL_TOKEN` | Your tunnel token (Cloudflare Tunnel only) |
| `RMM_RUSTDESK_HOST` | The RustDesk name, e.g. `rustdesk.example.com`; defaults to `RMM_DOMAIN` |
| `POSTGRES_PASSWORD` | A random password: `openssl rand -hex 24` |
| `RMM_ADMIN_EMAIL` | Your email; it becomes the first administrator |
| `RMM_ADMIN_PASSWORD` | Leave empty to have one generated (see step 4) |

Keep `RMM_UPDATE_PUBKEY` and `RMM_RELEASE_GITHUB_REPO` as they are. With
them, the server downloads official signed agent builds and agents verify
every update against that key.

## 3. Start it

```sh
docker compose pull
docker compose up -d
docker compose ps
```

Postgres and RustyRMM should show as `healthy` within a minute. Check the
console answers:

```sh
curl -s https://rmm.example.com/api/health
# {"agents_online":0,"ok":true,"version":"…"}
```

## 4. Sign in

Open `https://rmm.example.com` and sign in with `RMM_ADMIN_EMAIL`. If you
left `RMM_ADMIN_PASSWORD` empty, the generated password is in the log (it is
printed once):

```sh
docker compose logs rustyrmm | grep "created initial superuser"
```

Then:

1. **Profile:** change the password and turn on two-factor authentication.
2. **Settings → Email (SMTP):** add your mail server and send a test. Alert
   emails, scheduled reports and password resets need it.
3. **Settings → Agent updates → Sync now:** downloads the latest agent builds.
4. **Settings → Remote desktop:** shows **Ready** when the `rustdesk` profile
   is running.

## 5. Add your first device

Go to **Tenants & Sites**, create a tenant (a customer), open its site and
choose **Install**. Run the command it shows on the device:

```sh
# macOS / Linux (as root)
curl -fsSL https://rmm.example.com/install.sh | sudo sh -s -- <CODE>
```

```powershell
# Windows (elevated PowerShell)
& ([scriptblock]::Create((irm https://rmm.example.com/install.ps1))) -Code <CODE>
```

The device appears under **Devices** within a few seconds.

## Updating

```sh
docker compose pull
docker compose up -d
```

Database changes are applied automatically when the new version starts.
Agents update themselves to the version their tenant's update channel points
at.

To stay on one version, set `RMM_IMAGE=smugzombie/rustyrmm:0.9.1` (or another
version) in `.env`.

## Backups

Everything lives in Docker volumes. Back up the database regularly:

```sh
docker compose exec -T postgres pg_dump -U rmm rmm | gzip > rustyrmm-$(date +%F).sql.gz
```

Also keep a copy of `.env`, and of the `rustdesk` volume if you use remote
desktop: it holds the RustDesk server's key, and agents trust that key.
Treat backups as secrets.

## Documentation

The full documentation is served by your own server at
`https://rmm.example.com/docs/`. It covers configuration, Cloudflare Tunnel
in detail, agent installation, alerts, reports and troubleshooting.
