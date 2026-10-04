# Copyparty Fileserver Kit

This Docker setup provides a production-ready Copyparty deployment, optimized for security and resource management.

## Features

- Copyparty fileserver service
- Nginx reverse proxy with HTTPS (self-signed certificates)
- HTTP to HTTPS automatic redirect
- Optional internet access through a Cloudflare Tunnel (`cloudflared` profile), without opening any inbound ports
- Security hardened: no-new-privileges, AppArmor, dropped capabilities
- Minimal added capabilities
- Structured logging: JSON logging with rotation (10MB max, 5 files)
- Resource limits: CPU/memory constraints for all services
- Health checks for all services

## Directory Structure

```text
.
├── docker-compose.yml            # Service orchestration
├── .env                          # Variable configuration (e.g. NGINX_PORT)
├── cloudflared/
│   └── tunnel.env.example        # Cloudflare Tunnel token (copy to tunnel.env)
├── config/
│   ├── copyparty.conf.example    # Copyparty configuration (adjust this!)
├── nginx/
│   ├── nginx.conf                # Nginx configuration
│   └── certs/
│       ├── server.crt            # SSL certificate
│       └── server.key            # SSL private key
├── LICENSE                       # License file
└── README.md                     # This file
```

## Setup Instructions

### 1. Clone the repository

```bash
git clone https://github.com/barrax63/fileserver-kit.git
cd fileserver-kit
```

### 2. Generate self-signed certificates

```bash
mkdir -p nginx/certs

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/server.key \
  -out nginx/certs/server.crt \
  -subj "/C=US/ST=State/L=City/O=Organization/CN=copyparty.fritz.box" \
  -addext "subjectAltName=DNS:copyparty.fritz.box,DNS:localhost,IP:127.0.0.1"

chmod 644 nginx/certs/server.key
chmod 644 nginx/certs/server.crt
```

### 3. Adjust configuration

```bash
cd config
mv copyparty.conf.example copyparty.conf

# Adjust for your needs (e.g. create user accounts)
vi copyparty.conf
```

**Important:** `copyparty.conf.example` ships with a placeholder
`admin: CHANGE_ME_TO_A_STRONG_PASSWORD` account. Replace this with a real
username and a strong, unique password before exposing the service to
your network.

### 3b. (Optional) Change the published port

By default `docker-compose.yml` publishes nginx's port 8443. `.env`
already overrides this to `NGINX_PORT=4098`, i.e. the service is reachable
on **host port 4098** unless you change `.env`:

```bash
# .env
NGINX_PORT=4098
```

Adjust `NGINX_PORT` in `.env` to whichever host port you want to expose
(no `docker-compose.yml` changes needed).

### 3c. (Optional) Enable the Cloudflare Tunnel (internet access)

The `cloudflared` service publishes the stack on the internet through a
[Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/).
It only opens outbound connections to Cloudflare (port `7844`), so no
port forwarding or public IP is needed. You need a Cloudflare account
with a domain managed by Cloudflare.

The service is off by default: it belongs to the compose profile
`cloudflared` and is only part of the stack while that profile is
enabled. Skip this step to keep the stack LAN-only.

1. In the Cloudflare dashboard, go to **Networking** > **Tunnels** and
   create a tunnel. Copy its token: the long `eyJ…` string in the install
   command the dashboard shows. Do not run that command — the
   `cloudflared` container takes its place.

2. Store the token in `cloudflared/tunnel.env` (gitignored):

   ```bash
   cp cloudflared/tunnel.env.example cloudflared/tunnel.env
   chmod 600 cloudflared/tunnel.env
   vi cloudflared/tunnel.env
   ```

3. Enable the profile in `.env`, so that every `docker compose` command
   includes the tunnel:

   ```bash
   # .env
   COMPOSE_PROFILES=cloudflared
   ```

   Alternatively, pass `--profile cloudflared` to each command instead
   (e.g. `docker compose --profile cloudflared up -d`, and likewise for
   `down`, `pull` and `logs`).

   While the profile is enabled, `docker compose` refuses to load the
   stack if `cloudflared/tunnel.env` is missing.

4. Start the services as described in "4. Start the services" below and
   check the connector with `docker compose logs -f cloudflared`. Once the dashboard shows the
   tunnel as connected, open its **Routes** tab, select **Add route** >
   **Published application**, choose the public hostname and set
   **Service URL** to `http://nginx:8080`.

`nginx:8080` is a plain-HTTP listener that is only reachable inside the
docker network. It is the only place where nginx trusts Cloudflare's
`CF-Connecting-IP` header, so rate limits, logs and copyparty's IP bans
see the real visitor address. Pointing the tunnel at `copyparty:3923` or
at the HTTPS port instead would bypass this.

**Before going public**, review the `accs` sections in `copyparty.conf`:
the example grants anonymous visitors read-write access (`rw: *`), which
through the tunnel means anyone on the internet. Restrict access to
named accounts, and consider putting
[Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/self-hosted-public-app/)
in front of the hostname.

Cloudflare rejects request bodies over 100 MB on Free and Pro plans.
copyparty's web uploader (up2k) is unaffected, because it uploads in
chunks below that limit by default (`u2sz`), but larger single-request
uploads (e.g. `curl -T`, WebDAV clients) only work via the LAN address.

### 4. Start the services

```bash
docker compose up -d

# Follow logs
docker compose logs -f copyparty
docker compose logs -f nginx
```

### 5. Access the service

Open in browser: `https://<your-host>:4098` (replace `<your-host>` with
the IP/hostname of the machine running the stack, e.g.
`https://copyparty.fritz.box:4098` if you used the example certificate
`CN`/SAN and `4098` if you kept the default `NGINX_PORT` from `.env`;
see step 3b above if you changed the port).

Since the certificate is self-signed, your browser will show a security
warning on first visit — this is expected.

If you enabled the tunnel (step 3c), the service is also reachable from
the internet at the public hostname you routed, e.g.
`https://files.example.com`. Cloudflare serves it with a publicly trusted
certificate, so there is no warning.

## Maintenance

### Update

The images track rolling tags (`copyparty/ac:latest`,
`nginx:stable-alpine`, `cloudflare/cloudflared:latest`), so updating
pulls the newest build:

```bash
git pull
docker compose pull
docker compose up -d
```

### Restart

```bash
docker compose restart copyparty nginx
```

### View logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f copyparty
docker compose logs -f nginx
```

## Security Considerations

1. **Dropped Capabilities**: Services use `cap_drop: [ALL]` by default; only the specific capabilities each service actually needs are re-added (see comments in `docker-compose.yml`).
2. **AppArmor**: Default Docker AppArmor profile is enforced.
3. **No New Privileges**: Prevents privilege escalation in all containers.
4. **Least-privilege workers**: copyparty runs as uid `1000`. nginx's master process starts as root (with only `CHOWN`/`SETUID`/`SETGID` capabilities) but drops its worker processes to the unprivileged `nginx` user. cloudflared runs as uid `65532` with no added capabilities.
5. **Read-only root filesystem**: All containers run with `read_only: true`; the only writable paths are explicit `tmpfs` mounts (`/tmp`) and the bind-mounted data/config volumes.
6. **Config/certs mounted read-only**: nginx's `nginx.conf` and TLS certs are mounted `:ro`. The copyparty `./config:/cfg` mount is intentionally writable — copyparty persists an auto-generated security salt there (see comment in `docker-compose.yml`); making it read-only would invalidate shared links on every restart.
7. **Resource Limits**: CPU/memory limits and reservations are set for all services.
8. **TLS 1.2/1.3**: Modern TLS protocols with secure cipher suites.
9. **Security Headers**: X-Frame-Options, X-Content-Type-Options, HSTS enabled.
10. **Rate & connection limiting**: nginx limits requests/sec and concurrent connections per client IP.
11. **Change default credentials**: `config/copyparty.conf.example` ships with a placeholder account (`admin` / `CHANGE_ME_TO_A_STRONG_PASSWORD`) — always replace it before deploying.
12. **Cloudflare Tunnel (optional)**: Off unless the `cloudflared` profile is enabled. cloudflared publishes no ports and only connects outbound. Its token lives in the gitignored `cloudflared/tunnel.env`, never in the committed `.env`. nginx honours the `CF-Connecting-IP` header only on its internal tunnel listener, so clients on the published port cannot spoof their address. Anything copyparty serves anonymously is public once the tunnel is routed (see step 3c).
