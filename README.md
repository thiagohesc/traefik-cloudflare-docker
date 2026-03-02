# Traefik Reverse Proxy (Docker)

Central Traefik reverse proxy for Docker environments with automatic TLS via Cloudflare DNS-01.

---

## Summary

- Provides HTTP and HTTPS routing for Docker services
- Issues and renews TLS certificates automatically
- Uses Cloudflare DNS-01 challenge

---

## Overview

- Traefik runs as a single reverse proxy container
- Docker provider is enabled with `exposedByDefault=false`
- TLS certificates are stored in `letsencrypt/acme.json`
- Dashboard is enabled and bound to localhost only

---

## Requirements

- Docker
- Docker Compose
- A domain managed in Cloudflare
- A Cloudflare API token with DNS edit permissions
- External Docker network created in advance

---

## Quick Start

```bash
docker network create traefik-net
cp env.example .env
docker compose up -d
```

Expected result: Traefik is running and ready to issue certificates on first routed request.

---

## Configuration

- `docker-compose.yml` defines the Traefik service, ports, and entrypoints
- `.env` provides email, Cloudflare token, and network name
- `letsencrypt/acme.json` stores ACME account and certificate data

---

## Environment Variables

| Name             | Required | Default | Description                                 |
| ---------------- | -------- | ------- | ------------------------------------------- |
| ADMIN_EMAIL      | yes      | -       | Email used by ACME for certificate issuance |
| CF_DNS_API_TOKEN | yes      | -       | Cloudflare API token for DNS-01 challenges  |
| TRAEFIK_NET      | yes      | -       | External Docker network name                |

---

## Deployment

1. Create the external network listed in `TRAEFIK_NET`.
2. Set the required environment variables in `.env`.
3. Start the stack with Docker Compose.

---

## Operations

```bash
docker compose up -d
docker compose logs -f traefik
docker compose down
```

---

## Troubleshooting

- TLS challenge fails: check `CF_DNS_API_TOKEN` permissions and DNS propagation.
- `acme.json` permission errors: ensure it is `600` and mounted as `./letsencrypt`.
- Network errors: confirm the external network exists and matches `TRAEFIK_NET`.
- Dashboard not reachable: it is bound to `127.0.0.1:17000` only (`8080` is internal to the container).
