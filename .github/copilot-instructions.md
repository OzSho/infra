# Infrastructure Repository - AI Agent Instructions

> **Purpose:** Guidelines for AI coding agents (GitHub Copilot, Claude, Cursor, etc.) to safely modify this infrastructure repository.
> **Version:** 1.0.0

---

## 1. Repository Overview

This repository contains Docker Compose configurations for a self-hosted infrastructure stack with a **VPN-first security model**.

### Architecture Principles

1. **Single Entry Point** — Only WireGuard VPN (UDP 51820) is exposed to the internet
2. **Reverse Proxy** — Traefik handles internal routing and TLS
3. **Centralized Auth** — Authentik SSO protects all services via forward auth
4. **On-Demand Scaling** — Sablier can start stopped containers on first request
5. **Infrastructure as Code** — All configurations are declarative YAML

### Service Dependency Order

```
1. wg-easy      → VPN gateway (external access point)
2. traefik      → Reverse proxy (routes all HTTP traffic)
3. authentik    → SSO provider (protects services)
4. sablier      → On-demand scaling (optional)
5. *            → All other services (any order)
```

---

## 2. File Patterns & Conventions

### Directory Structure (Per Service)

```
service-name/
 compose.yaml      # Docker Compose definition (required)
 .env.example      # Environment template (if secrets needed)
 README.md         # Service documentation (optional)
 config/           # Configuration files (if needed)
```

### Naming Conventions

| Item | Convention | Example |
|------|------------|---------|
| Folder name | lowercase, hyphenated | `wg-easy`, `adguardhome` |
| Container name | Same as folder | `container_name: wg-easy` |
| Compose file | `compose.yaml` | Not `docker-compose.yml` |
| Network | Always `backend` | External network |

### Docker Compose Structure

Use this ordering within compose files:

```yaml
services:
  service-name:
    image: registry/image:specific-tag    # Pin versions, avoid :latest
    container_name: service-name          # Match folder name
    restart: unless-stopped               # Default restart policy
    
    environment:
      - TZ=${TZ:-UTC}                     # Timezone
      - SECRET=${SECRET}                  # Reference .env variables
    
    volumes:
      - ./data:/data                      # Relative paths preferred
    
    labels:
      # Traefik labels (see below)
    
    networks:
      - backend

networks:
  backend:
    external: true
```

---

## 3. Traefik Label Patterns

### Standard Service (with Authentik)

```yaml
labels:
  # Enable Traefik
  - "traefik.enable=true"
  
  # Router
  - "traefik.http.routers.SERVICE.rule=Host(`service.example.com`)"
  - "traefik.http.routers.SERVICE.entrypoints=websecure"
  - "traefik.http.routers.SERVICE.tls=true"
  - "traefik.http.routers.SERVICE.tls.certresolver=letsencrypt"
  
  # Service (internal port)
  - "traefik.http.routers.SERVICE.service=SERVICE-svc"
  - "traefik.http.services.SERVICE-svc.loadbalancer.server.port=PORT"
  
  # Middlewares (standard chain)
  - "traefik.http.routers.SERVICE.middlewares=authentik@file,security-headers@file,ratelimit@file,ip-whitelist@file"
```

### Service WITHOUT Authentik (OAuth2 apps)

Some services handle their own OAuth2 flow and will break with forward auth:

```yaml
labels:
  # ... router config ...
  # Note: authentik@file REMOVED to allow OAuth2 callbacks
  - "traefik.http.routers.SERVICE.middlewares=security-headers@file,ratelimit@file,ip-whitelist@file"
```

### Sablier On-Demand Service

For services that should start on first request:

```yaml
services:
  service-name:
    restart: "no"  # Required for Sablier
    labels:
      # Standard Traefik config...
      
      # Allow routing to stopped containers
      - "traefik.docker.allownonrunning=true"
      
      # Sablier middleware
      - "traefik.http.routers.SERVICE.middlewares=sablier-SERVICE,...other-middlewares..."
      
      # Sablier middleware config
      - "traefik.http.middlewares.sablier-SERVICE.plugin.sablier.group=SERVICE"
      - "traefik.http.middlewares.sablier-SERVICE.plugin.sablier.sessionDuration=30m"
      - "traefik.http.middlewares.sablier-SERVICE.plugin.sablier.sablierUrl=http://sablier:10000"
      - "traefik.http.middlewares.sablier-SERVICE.plugin.sablier.dynamic.displayName=Service Name"
      - "traefik.http.middlewares.sablier-SERVICE.plugin.sablier.dynamic.theme=hacker-terminal"
      
      # Sablier control
      - "sablier.enable=true"
      - "sablier.group=SERVICE"
```

### Homepage Dashboard Integration

```yaml
labels:
  - "homepage.group=Category"           # Networking, Security, Storage, etc.
  - "homepage.name=Service Name"        # Display name
  - "homepage.icon=service.png"         # Icon from dashboard-icons
  - "homepage.href=https://service.example.com"
  - "homepage.description=Short description"
```

---

## 4. Security Rules (NEVER VIOLATE)

### Hard Rules

```
1. NEVER commit secrets, passwords, or API keys
2. NEVER use :latest tags without explicit user approval
3. NEVER remove security middlewares without justification
4. NEVER expose ports to host except WireGuard (51820/udp)
5. ALWAYS use the external `backend` network
6. ALWAYS use environment variables for secrets
7. ALWAYS create .env.example for services with secrets
```

### Secrets Pattern

```yaml
# CORRECT - Reference environment variable
environment:
  - DATABASE_PASSWORD=${DATABASE_PASSWORD}

# WRONG - Never hardcode
environment:
  - DATABASE_PASSWORD=actual-password-here
```

### .env.example Format

```bash
# Service Name Configuration
# Copy to .env and fill in values

# Database
POSTGRES_USER=servicename
POSTGRES_PASSWORD=<generate-secure-password>
POSTGRES_DB=servicename

# API Keys (from provider dashboard)
API_KEY=<your-api-key>
```

---

## 5. Adding a New Service

### Step-by-Step Checklist

1. **Create directory**: `mkdir service-name`

2. **Create compose.yaml** with:
   - [ ] Pinned image version (not `:latest`)
   - [ ] `container_name` matching folder
   - [ ] `restart: unless-stopped`
   - [ ] `networks: [backend]`
   - [ ] Traefik labels with all middlewares
   - [ ] Homepage labels

3. **Create .env.example** if service needs secrets

4. **Verify configuration**:
   ```bash
   cd service-name
   docker compose config  # Syntax check
   ```

5. **Test deployment**:
   ```bash
   docker compose up      # With logs visible
   # If successful:
   docker compose up -d   # Detached
   ```

### Template for New Service

```yaml
services:
  new-service:
    image: registry/image:1.0.0
    container_name: new-service
    restart: unless-stopped
    
    environment:
      - TZ=${TZ:-UTC}
    
    volumes:
      - ./data:/data
    
    labels:
      # Traefik
      - "traefik.enable=true"
      - "traefik.http.routers.new-service.rule=Host(`new-service.example.com`)"
      - "traefik.http.routers.new-service.entrypoints=websecure"
      - "traefik.http.routers.new-service.tls=true"
      - "traefik.http.routers.new-service.tls.certresolver=letsencrypt"
      - "traefik.http.routers.new-service.service=new-service-svc"
      - "traefik.http.services.new-service-svc.loadbalancer.server.port=8080"
      - "traefik.http.routers.new-service.middlewares=authentik@file,security-headers@file,ratelimit@file,ip-whitelist@file"
      
      # Homepage
      - "homepage.group=Category"
      - "homepage.name=New Service"
      - "homepage.icon=new-service.png"
      - "homepage.href=https://new-service.example.com"
      - "homepage.description=Service description"
    
    networks:
      - backend

networks:
  backend:
    external: true
```

---

## 6. Modifying Existing Services

### Pre-Modification Checklist

- [ ] Read the current compose file completely
- [ ] Understand what each label and volume does
- [ ] Check for dependent services
- [ ] Identify secrets that might need updating

### Safe Modification Process

```bash
# 1. Validate syntax after changes
docker compose config

# 2. Apply with logs visible
docker compose up

# 3. If successful, run detached
docker compose up -d

# 4. Verify service is working
docker compose logs -f
curl -I https://service.example.com
```

### Rollback

```bash
# If something breaks, revert via git
git checkout -- compose.yaml
docker compose up -d
```

---

## 7. Middleware Reference

| Middleware | Purpose | When to Use |
|------------|---------|-------------|
| `authentik@file` | SSO forward auth | Most services |
| `security-headers@file` | HSTS, CSP, etc. | Always |
| `ratelimit@file` | Request throttling | Always |
| `ip-whitelist@file` | VPN subnet only | Always |
| `sablier-*` | On-demand start | Idle services |

### When to OMIT `authentik@file`

- Services with built-in OAuth2 (blocks callbacks)
- API endpoints that need external access
- Services that handle their own auth

Always document the reason as a comment:

```yaml
# Note: authentik@file removed - service handles OAuth2 internally
- "traefik.http.routers.service.middlewares=security-headers@file,ratelimit@file"
```

---

## 8. Common Operations

### Start All Services

```bash
for dir in traefik authentik wg-easy forgejo vaultwarden seafile homepage duplicati portainer adguardhome sablier ddclient; do
  [ -d "$dir" ] && (cd "$dir" && docker compose up -d)
done
```

### Update All Services

```bash
for dir in */; do
  [ -f "$dir/compose.yaml" ] && (cd "$dir" && docker compose pull && docker compose up -d)
done
```

### View Logs

```bash
cd service-name
docker compose logs -f
```

### Check Health

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

---

## 9. Quality Checklist

Before submitting changes, verify:

- [ ] Image tag is pinned (not `:latest`)
- [ ] Container name matches folder name
- [ ] Uses `backend` external network
- [ ] Traefik labels follow standard pattern
- [ ] Security middlewares included
- [ ] No hardcoded secrets
- [ ] `.env.example` exists if secrets needed
- [ ] Syntax validated with `docker compose config`

---

## 10. Reference

### Available Middlewares (traefik/dynamic.yml)

- `authentik@file` — Forward auth to Authentik
- `security-headers@file` — Security response headers
- `ratelimit@file` — Rate limiting
- `ip-whitelist@file` — IP allowlist
`redirect-to-https@file` - HTTP→HTTPS redirect 

### Network

- Name: `backend`
- Type: External (create with `docker network create backend`)

### Placeholders to Replace

- `example.com` → Your domain
- `letsencrypt` → Your cert resolver name
- Port numbers → Service-specific ports

---

*This document enables AI agents to safely modify infrastructure configurations while maintaining security and consistency standards.*
