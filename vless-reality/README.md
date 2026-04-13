# VLESS + REALITY via 3X-UI

Self-hosted VPN using [3X-UI](https://github.com/MHSanaei/3x-ui) (Xray panel) with VLESS+REALITY transport and DuckDNS for dynamic DNS.

REALITY mimics a real TLS handshake against a legitimate destination (e.g. `www.microsoft.com`), making traffic indistinguishable from regular HTTPS.

## Requirements

- VPS with a public IP
- Docker + Docker Compose
- A [DuckDNS](https://www.duckdns.org) account

## Setup

```bash
cp .env.example .env
# fill in TZ, DUCKDNS_SUBDOMAIN, DUCKDNS_TOKEN
docker compose up -d
```

**First-time panel config:**

1. Open `http://localhost:2053` (SSH tunnel if remote: `ssh -L 2053:localhost:2053 user@server`)
2. Login: `admin` / `admin` — **change immediately** in Settings → User
3. Inbounds → Add Inbound:
   - Protocol: `vless`
   - Port: `8443`
   - Security: `reality`
   - Dest (uTLS): `www.microsoft.com:443`
   - SNI: `www.microsoft.com`
   - Generate keys, save
4. Copy the share link / QR code to your client

## Ports

| Port | Purpose |
|------|---------|
| `2053` | 3X-UI admin panel (bound to `127.0.0.1`) |
| `8443` | VLESS+REALITY inbound (public) |

## Clients

- **iOS/macOS**: [Streisand](https://apps.apple.com/app/streisand/id6450534064)
- **Android**: [v2rayNG](https://github.com/2dust/v2rayNG)
- **Windows/Linux**: [Hiddify](https://github.com/hiddify/hiddify-app)

## Notes

- Admin panel is bound to `127.0.0.1` only; access via SSH tunnel when managing remotely
- Pin image digests (already done) to avoid unexpected updates breaking the config
- 3X-UI persists all config in `./data/3x-ui` — back this up after initial setup
