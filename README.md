# RustyRMM self-hosted

Run your own [RustyRMM](https://rustyrmm.com) server with Docker Compose: the
RustyRMM server and web console, a Postgres database, and HTTPS through
Cloudflare Tunnel, Let's Encrypt, or a reverse proxy or tunnel you already run. Remote desktop is built in (the **Screen**
tab on each device), so nothing extra is needed for it; a RustDesk server is
available as an optional add-on.

## Requirements

- A Linux host with Docker Engine 24+ and Docker Compose v2. Ubuntu 24.04 is
  a good choice.
- At least 2 vCPU, 2 GB RAM and 20 GB disk.
- A domain name for the console, for example `rmm.example.com`, served over
  HTTPS.

## 1. Choose how the console is reached

| | Cloudflare Tunnel (default) | Let's Encrypt (Caddy) | Your own proxy or tunnel |
|---|---|---|---|
| `COMPOSE_PROFILES` | `tunnel` | `caddy` | empty |
| Inbound ports for the console | None | 80 and 443 | Whatever your proxy needs |
| Public IP / port forwarding | Not needed | Needed | Up to your setup |
| DNS | Domain on Cloudflare | Any provider: an `A` record pointing at this host | Pointing at your proxy or tunnel |

**Cloudflare Tunnel:** in the Cloudflare dashboard go to **Zero Trust →
Networks → Tunnels**, create a tunnel and copy its token. Add a public
hostname (for example `rmm.example.com`) with the service
`http://rustyrmm:8080`.

**Let's Encrypt:** point the domain's DNS record at this host before the
first start, and open ports 80 and 443.

**Your own proxy or tunnel** (nginx, Traefik, Caddy, HAProxy, Nginx Proxy
Manager, a `cloudflared` you already run, ...): the stack only starts
RustyRMM and Postgres, and your proxy forwards `https://rmm.example.com` to
the server's plain-HTTP port. See
[Use your own reverse proxy or tunnel](#use-your-own-reverse-proxy-or-tunnel)
for how to connect it and what the proxy must allow.

**Remote desktop:** built in. Sessions run through the console's normal
HTTPS connection, so no extra ports are needed.

**RustDesk add-on (optional):** only if your team prefers the RustDesk desktop
client. The `rustdesk` profile runs a RustDesk server next to RustyRMM. It
needs TCP 21115–21117 and UDP 21116 open to this host. With Cloudflare
Tunnel, give it its own DNS-only (grey cloud) record, for example
`rustdesk.example.com`, because tunnelled names don't carry those ports.

## 2. Get the files and configure

```sh
git clone https://github.com/RustyRmm/rustyrmm-selfhosted.git
cd rustyrmm-selfhosted
cp sample.env .env
```

Edit `.env`:

| Setting | What to put |
|---|---|
| `COMPOSE_PROFILES` | `tunnel` (Cloudflare Tunnel), `caddy` (Let's Encrypt), or empty (your own proxy or tunnel); add `rustdesk` only for the optional RustDesk add-on |
| `COMPOSE_FILE` | Your own proxy only: `docker-compose.yml:expose.yml` to publish the server's port on this host (see [below](#use-your-own-reverse-proxy-or-tunnel)) |
| `RMM_DOMAIN` | The console's name, e.g. `rmm.example.com`. Browsers and agents must reach it at exactly `https://RMM_DOMAIN` |
| `CLOUDFLARE_TUNNEL_TOKEN` | Your tunnel token (Cloudflare Tunnel only) |
| `RMM_RUSTDESK_HOST` | RustDesk add-on only: its name, e.g. `rustdesk.example.com`; defaults to `RMM_DOMAIN` |
| `POSTGRES_PASSWORD` | A random password: `openssl rand -hex 24` |
| `RMM_ADMIN_EMAIL` | Your email; it becomes the first administrator |
| `RMM_ADMIN_PASSWORD` | Leave empty to have one generated (see step 4) |
| `RMM_SECRET_KEY` | Only if you'll back up to S3 from **Settings → Backups**: `openssl rand -hex 32`, and keep it unchanged |

Keep `RMM_UPDATE_PUBKEY` and `RMM_RELEASE_GITHUB_REPO` as they are. With
them, the server downloads official signed agent builds and agents verify
every update against that key. The same key checks the signed vulnerability
feed (the **Vulnerabilities** page); `RMM_VULN_FEED_URL` points it at a mirror,
or `off` turns vulnerability monitoring off.

## 3. Start it

```sh
docker compose pull
docker compose up -d
docker compose ps
```

Postgres and RustyRMM should show as `healthy` within a minute. With your own
proxy, check the server first on the published port
(`curl -s http://127.0.0.1:8080/api/health`), then through the proxy. Check
the console answers:

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
4. **Settings → Remote desktop:** the built-in remote view is on for new
   installs, and each session asks the person at the device first. Check the
   options suit you. (With the `rustdesk` add-on, the RustDesk section shows
   **Ready** once its server is running.)

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

## Use your own reverse proxy or tunnel

<!-- Kept in sync with docs-site/docs/deployment/reverse-proxy.md in
     github.com/RustyRmm/rustyrmm (the docs your server serves at /docs). -->

If you already run a reverse proxy or tunnel, skip the bundled Caddy and
Cloudflare Tunnel: set `COMPOSE_PROFILES=` (empty, or just `rustdesk`) and
point your proxy at the RustyRMM server. The server speaks plain HTTP on port
8080; your proxy terminates TLS for `https://RMM_DOMAIN`.

### Connect the proxy

Pick one:

- **Publish the port on this host** (proxy on this host, another machine, or
  a tunnel agent such as `cloudflared` running outside Docker). In `.env`:

  ```sh
  COMPOSE_FILE=docker-compose.yml:expose.yml
  RMM_HTTP_BIND=127.0.0.1   # this host's LAN address if the proxy is elsewhere
  RMM_HTTP_PORT=8080
  ```

  Then `docker compose up -d` and use `http://<this host>:8080` as the
  upstream. When binding to a LAN address, firewall the port so that only the
  proxy can reach it (see [client addresses](#what-the-proxy-must-allow)).

- **Share the Docker network** (proxy running as a container on this host,
  e.g. Traefik or Nginx Proxy Manager). Nothing is published; attach the proxy
  to this stack's network and use `http://rustyrmm:8080` as the upstream. In
  the proxy's compose file:

  ```yaml
  services:
    proxy:
      networks: [default, rustyrmm]
  networks:
    rustyrmm:
      name: rustyrmm_default
      external: true
  ```

  For Traefik, label the `rustyrmm` service (for example in a
  `docker-compose.override.yml` next to `docker-compose.yml`) with your
  router rule and `traefik.http.services.rustyrmm.loadbalancer.server.port=8080`.

### What the proxy must allow

| Requirement | Why | Example |
|---|---|---|
| HTTPS at exactly `https://RMM_DOMAIN` | Sign-in cookies are `Secure`, agents connect to this URL, and browsers' WebSocket `Origin` must match it | One certificate for `RMM_DOMAIN` |
| WebSocket upgrades on every path | Agents keep a WebSocket to `/api/agent/ws`; the terminal and remote view use WebSockets too | nginx: `proxy_http_version 1.1`, `Upgrade` and `Connection` headers; Nginx Proxy Manager: **Websockets Support** |
| Pass the `Host` header through | The server accepts a browser WebSocket when its origin matches the `Host` or `RMM_DOMAIN` | nginx: `proxy_set_header Host $host` |
| Long idle timeouts (1 hour or more) | A terminal or remote session can sit idle; short read timeouts cut it off | nginx: `proxy_read_timeout 3600s` and `proxy_send_timeout 3600s` |
| Large request bodies, not buffered | File uploads to devices and the file library can be up to 4 GB | nginx: `client_max_body_size 0`, `proxy_request_buffering off`, `proxy_buffering off` |
| Real client address in `X-Forwarded-For` | Audit log, sign-in rate limiting and per-user sign-in allowlists use it | nginx: `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for` |
| No extra sign-in in front of agent paths | Agents and install commands can't pass SSO or access gates | Exempt `/api/agent/`, `/install.sh` and `/install.ps1` from forward-auth, Cloudflare Access and bot challenges |

**Client addresses:** RustyRMM trusts `X-Forwarded-For` only when the
connection comes from a loopback or private address (your proxy), and uses
the **last** entry, the address your proxy saw. With several proxies in a
row, make the last one put the real client address last (behind Cloudflare,
for example, restore it from `CF-Connecting-IP` first). Because private
addresses are trusted, don't let other machines reach the published port
directly.

**Remote view's direct path** (optional, off by default) connects browser and
device peer to peer over WebRTC, so it needs nothing from your proxy.

### Examples

nginx (TLS settings omitted):

```nginx
server {
    listen 443 ssl;
    server_name rmm.example.com;
    # ssl_certificate / ssl_certificate_key ...

    client_max_body_size 0;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_request_buffering off;
        proxy_buffering off;
    }
}

# In the http block:
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

Caddy you already run (WebSockets, `Host` and `X-Forwarded-For` work out of
the box):

```caddyfile
rmm.example.com {
	reverse_proxy 127.0.0.1:8080 {
		flush_interval -1
	}
}
```

Your own Cloudflare Tunnel: add a public hostname for `RMM_DOMAIN` with the
service `http://<this host>:8080`, or `http://rustyrmm:8080` if `cloudflared`
shares the `rustyrmm_default` network. Cloudflare limits uploads to 100 MB on
most plans, and its Bot Fight Mode and Access must not challenge agent paths
(see `/docs/deployment/cloudflare-tunnel` on your server).

### Check it

```sh
curl -s https://rmm.example.com/api/health     # through the proxy
docker compose logs rustyrmm | grep -i "rejected cross-origin"
```

The second command should print nothing. If it shows lines, the proxy
changes the `Host` header or `RMM_DOMAIN` doesn't match the address in the
browser. After installing an agent, the device should show as online within
seconds. If it doesn't, WebSocket upgrades aren't reaching the server.

| Symptom | Likely cause |
|---|---|
| Sign-in works but you're signed straight out, or nothing saves | The console is served over plain HTTP, or at a name other than `RMM_DOMAIN` |
| Devices flap between online and offline; terminal or remote view disconnect after about a minute | Proxy idle timeout too short, or WebSocket upgrades not forwarded |
| Uploads fail with 413 | Request body limit on the proxy (nginx's default is 1 MB) or the tunnel provider |
| Audit log shows the proxy's address for every sign-in | The proxy isn't setting `X-Forwarded-For`, or it connects from a public address |
| `install.sh` returns a login page or HTML | An SSO, forward-auth or bot challenge is in front of the agent paths |

## Updating

```sh
docker compose pull
docker compose up -d
```

Database changes are applied automatically when the new version starts.
Agents update themselves to the version their tenant's update channel points
at.

To stay on one version, set `RMM_IMAGE=smugzombie/rustyrmm:0.27.0` (or another
version) in `.env`.

**Existing installs using RustDesk:** `git pull` won't touch your `.env`, so
the `rustdesk` profile keeps running. To drop it, switch devices to the
built-in remote view (step 4 above), remove `rustdesk` from
`COMPOSE_PROFILES`, then `docker compose up -d --remove-orphans`. Keep the
`rustdesk` volume if you might turn it back on.

## Backups

Everything lives in Docker volumes. Back up the database and the `data`
volume (the file library and agent builds) regularly:

```sh
docker compose exec -T postgres pg_dump -U rmm -Fc rmm > rustyrmm-$(date +%F).dump
docker run --rm -v rustyrmm_data:/data -v "$PWD":/out alpine:3 \
  tar czf /out/rustyrmm-data-$(date +%F).tgz /data
```

Also keep a copy of `.env`, and of the `rustdesk` volume if you use the
RustDesk add-on: it holds the RustDesk server's key, and agents trust that key.
The `caddy` volume only holds certificates, which are issued again
automatically. Treat backups as secrets: they contain password hashes and
agent credentials.

## Documentation

The full documentation is served by your own server at
`https://rmm.example.com/docs/`. It covers configuration, Cloudflare Tunnel
in detail, restoring backups, agent installation, remote desktop, alerts, reports and
troubleshooting.
