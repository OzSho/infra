# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in this repository, please report it responsibly:

1. you can create a public GitHub issue with solution if possible.
2. Contact the maintainer directly

## Security Best Practices

This repository demonstrates security-conscious infrastructure patterns:

### Secrets Management
- All credentials are stored in `.env` files (gitignored)
- Only `.env.example` templates are committed
- Sensitive files have restricted permissions (`chmod 600`)

### Network Security
- Services communicate via isolated Docker network
- Only Traefik exposes ports to the host
- Internal services are not directly accessible

### Authentication
- Centralized SSO via Authentik forward auth
- Rate limiting on all endpoints
- Security headers (HSTS, CSP) enforced

### TLS/SSL
- Automatic certificate provisioning via Let's Encrypt
- HTTP automatically redirected to HTTPS
- Strong cipher suites via Traefik defaults

## Supported Versions

This is a personal infrastructure repository. Security updates are applied as needed but no formal support schedule is maintained.
