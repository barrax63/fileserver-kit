# Copyparty Fileserver Kit

This Docker setup provides a production-ready Copyparty deployment, optimized for security and resource management.

## Features

- Copyparty fileserver service
- Nginx reverse proxy with HTTPS (self-signed certificates)
- HTTP to HTTPS automatic redirect
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

## Maintenance

### Update

The images track rolling tags (`copyparty/ac:latest`,
`nginxinc/nginx-unprivileged:stable-alpine`), so updating pulls the
newest build:

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
4. **Least-privilege workers**: copyparty runs as uid `1000`. nginx's master process starts as root (with only `CHOWN`/`SETUID`/`SETGID` capabilities) but drops its worker processes to the unprivileged `nginx` user.
5. **Read-only root filesystem**: Both containers run with `read_only: true`; the only writable paths are explicit `tmpfs` mounts (`/tmp`) and the bind-mounted data/config volumes.
6. **Config/certs mounted read-only**: nginx's `nginx.conf` and TLS certs are mounted `:ro`. The copyparty `./config:/cfg` mount is intentionally writable — copyparty persists an auto-generated security salt there (see comment in `docker-compose.yml`); making it read-only would invalidate shared links on every restart.
7. **Resource Limits**: CPU/memory limits and reservations are set for all services.
8. **TLS 1.2/1.3**: Modern TLS protocols with secure cipher suites.
9. **Security Headers**: X-Frame-Options, X-Content-Type-Options, HSTS enabled.
10. **Rate & connection limiting**: nginx limits requests/sec and concurrent connections per client IP.
11. **Change default credentials**: `config/copyparty.conf.example` ships with a placeholder account (`admin` / `CHANGE_ME_TO_A_STRONG_PASSWORD`) — always replace it before deploying.
