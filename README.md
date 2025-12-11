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

```
.
├── docker-compose.yml            # Service orchestration
├── .env                          # Varialbe configuration
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

### 4. Start the services

```bash
docker compose up -d

# Follow logs
docker compose logs -f copyparty
docker compose logs -f nginx
```

### 5. Access the service

Open in browser: `https://.fritz.box:5096`

## Maintenance

### Update

```bash
git pull
docker compose up -d --pull always
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

1. **Dropped Capabilities**: Services use `cap_drop: [ALL]` by default; only required capabilities are re-added.
2. **AppArmor**: Default Docker AppArmor profile is enforced.
3. **No New Privileges**: Prevents privilege escalation in all containers.
4. **Immutable Config**: copyparty and nginx configurations are mounted read-only.
5. **Resource Limits**: CPU/memory limits and reservations are set for all services.
6. **TLS 1.2/1.3**: Modern TLS protocols with secure cipher suites.
7. **Security Headers**: X-Frame-Options, X-Content-Type-Options, HSTS enabled.
