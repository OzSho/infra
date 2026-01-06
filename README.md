# Infrastructure

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![Traefik](https://img.shields.io/badge/Traefik-Proxy-24A1C1?logo=traefikproxy&logoColor=white)](https://traefik.io/)

Production-ready, self-hosted infrastructure stack featuring 12 containerized services with **VPN-first security model**, centralized authentication, automated TLS, and security hardening.

## Security Model: VPN-First Architecture

This infrastructure uses a **defense-in-depth approach** where the only publicly exposed port is the WireGuard VPN (UDP 51820). All other services, including the reverse proxy, are accessible only after establishing a VPN connection.

```
                             INTERNET
                                 │
                    ┌────────────┴────────────┐
                    │   FIREWALL (UDP 51820)  │
                    │    SINGLE OPEN PORT     │
                    └────────────┬────────────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │   WG-Easy     │
                         │ WireGuard VPN │
                         │    Gateway    │
                         └───────┬───────┘
                                 │
                    ════════════════════════
                        VPN TUNNEL ONLY
                    ════════════════════════
                                 │
                                 ▼
                       INTERNAL NETWORK
                                 │
                         ┌───────┴───────┐
                         │    Traefik    │
                         │ Reverse Proxy │
                         │  TLS + Router │
                         └───────┬───────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
       ┌──────▼─────┐    ┌──────▼──────┐   ┌──────▼──────┐
       │ Authentik  │    │   Sablier   │   │  AdGuard    │
       │    SSO     │    │  On-Demand  │   │    DNS      │
       │  Provider  │    │   Scaling   │   │  Blocking   │
       └──────┬─────┘    └─────────────┘   └─────────────┘
              │
    ┌─────────┼─────────┬─────────┬─────────┬─────────┐
    │         │         │         │         │         │
┌───▼───┐ ┌──▼───┐ ┌───▼────┐ ┌──▼──┐ ┌───▼────┐ ┌──▼───┐
│Forgejo│ │Vault │ │Seafile │ │Home │ │Duplicat│ │Portai│
│  Git  │ │warden│ │ Files  │ │page │ │Backups │ │ner   │
└───────┘ └──────┘ └────────┘ └─────┘ └────────┘ └──────┘

                 Docker Network: backend
```

### Why VPN-First?

| Traditional Approach | VPN-First Approach |
|---------------------|-------------------|
| Expose ports 80/443 to internet | Only UDP 51820 exposed |
| Rely on WAF/rate limiting | Attack surface reduced by 99% |
| Vulnerable to zero-days in web apps | Attackers can't reach web apps |
| DDoS targets web services | VPN protocol is DDoS-resistant |
| Must patch immediately | Defense in depth buys time |

## Services

| Service | Description | SSO | Forward Auth |
|---------|-------------|-----|--------------|
| **WG-Easy** | WireGuard VPN gateway (only external port) | — | ✓ |
| **Traefik** | Reverse proxy with automatic TLS | — | ✓ |
| **Authentik** | Identity provider & SSO | ✓ | — |
| **Forgejo** | Git repository hosting with CI/CD | ✓ | ✓ |
| **Vaultwarden** | Bitwarden-compatible password manager | ✓ | ✓ |
| **Seafile** | File sync & share platform | ✓ | ✓ |
| **Homepage** | Application dashboard | — | ✓ |
| **Duplicati** | Encrypted backup solution | — | ✓ |
| **Portainer** | Docker management UI | ✓ | ✓ |
| **AdGuard Home** | Network-wide DNS & ad blocking | — | ✓ |
| **Sablier** | On-demand container scaling | — | — |
| **ddclient** | Dynamic DNS updater | — | — |

## Features

### Security Layers
1. **VPN Gateway** — WireGuard is the only entry point from the internet
2. **Reverse Proxy** — Traefik handles TLS termination and routing
3. **Forward Auth** — Authentik SSO protects all applications
4. **Security Headers** — HSTS, CSP, X-Frame-Options on all responses
5. **Rate Limiting** — Request throttling prevents abuse
6. **IP Whitelisting** — Restrict to VPN subnet only
7. **Secrets Management** — All credentials via environment variables

### Infrastructure
- **Single Entry Point** — One UDP port to secure and monitor
- **On-Demand Scaling** — Sablier starts containers on first request
- **Health Checks** — Service dependency management
- **Automated Backups** — Encrypted backups with Duplicati
- **DNS-Level Blocking** — AdGuard Home for network protection

### Operations
- **Infrastructure as Code** — Declarative YAML configurations
- **AI-Agent Ready** — Includes copilot-instructions.md for AI assistance
- **Reproducible** — Full stack deployable from this repository

## Quick Start

### Prerequisites

- Docker Engine 24.0+
- Docker Compose v2.20+
- Domain with DNS access
- Linux host with WireGuard kernel module
- Static IP or Dynamic DNS configured

### 1. Clone Repository

```bash
git clone https://github.com/OzSho/infra.git
cd infra
```

### 2. Create Docker Network

```bash
docker network create backend
```

### 3. Configure Environment

```bash
# Copy all example files
for dir in */; do
  [ -f "$dir/.env.example" ] && cp "$dir/.env.example" "$dir/.env"
done

# Edit each .env file with your values
```

### 4. Start Services (Order Matters)

```bash
# 1. WireGuard VPN (external gateway)
cd wg-easy && docker compose up -d && cd ..

# 2. Traefik (reverse proxy)
cd traefik && docker compose up -d && cd ..

# 3. Authentik (SSO provider)
cd authentik && docker compose up -d && cd ..

# 4. Remaining services (any order)
for svc in forgejo vaultwarden seafile homepage duplicati portainer adguardhome sablier ddclient; do
  [ -d "$svc" ] && cd "$svc" && docker compose up -d && cd ..
done
```

### 5. Configure Firewall

```bash
# Allow only WireGuard from internet
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 51820/udp comment "WireGuard VPN"
sudo ufw allow from 10.8.0.0/24 comment "VPN clients"
sudo ufw enable
```

### 6. Connect via VPN

1. Access WG-Easy setup at `http://your-server-ip:51821` (temporary, before firewall)
2. Create your first VPN client
3. Import config to WireGuard client
4. Connect and access services via internal domains

## Configuration

### Domain Setup

Replace `example.com` throughout:

```bash
find . -name "*.yaml" -o -name "*.yml" | xargs sed -i 's/example\.com/yourdomain.com/g'
```

### Traefik Labels Pattern

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.SERVICE.rule=Host(`service.example.com`)"
  - "traefik.http.routers.SERVICE.entrypoints=websecure"
  - "traefik.http.routers.SERVICE.tls.certresolver=letsencrypt"
  - "traefik.http.routers.SERVICE.middlewares=authentik@file,security-headers@file"
  - "traefik.http.services.SERVICE.loadbalancer.server.port=PORT"
```

### Adding New Services

See [.github/copilot-instructions.md](.github/copilot-instructions.md) for detailed patterns and AI-assisted development guidelines.

## Directory Structure

```
infra/
 .github/
   └── copilot-instructions.md  # AI agent guidelines
 traefik/                      # Reverse proxy
 authentik/                    # SSO provider
 wg-easy/                      # VPN gateway
 forgejo/                      # Git hosting
 vaultwarden/                  # Passwords
 seafile/                      # File sync
 homepage/                     # Dashboard
 duplicati/                    # Backups
 portainer/                    # Docker UI
 adguardhome/                  # DNS
 sablier/                      # On-demand scaling
 ddclient/                     # Dynamic DNS
```

## Security Considerations

### Network Security
- **Only port 51820/udp is exposed** to the internet
- All web services accessible only through VPN tunnel
- Internal `backend` Docker network isolates containers
- Traefik binds to Docker network, not host ports 80/443

### Secrets Management
- **Never commit `.env` files**
- Use `.env.example` with placeholders
- Restrict file permissions: `chmod 600 .env`

### Backup Strategy
- Duplicati encrypts all backups at rest
- Store backups off-site (NAS, S3, B2)
- Test restore procedures regularly

## Contributing

Contributions welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) first.

## License

MIT License — see [LICENSE](LICENSE)

---

**Security Note:** This repository contains sanitized configurations. Replace all `example.com` references and generate unique secrets before deployment.
