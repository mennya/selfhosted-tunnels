# Marzban — Multi-node VPN Panel

Primary/secondary setup using [Marzban](https://github.com/Gozargah/Marzban) panel + [Marzban Node](https://github.com/Gozargah/Marzban-node). Users get inbounds on both servers in a single config — clients failover automatically.

## Architecture

```
Hetzner (panel/)          ← primary inbound + management panel
    │ TLS node API
Local machine (node/)     ← secondary inbound, auto-failover
```

## Deploy order

### 1. Panel on Hetzner

```bash
cd panel
cp .env.example .env
# fill SECRET_KEY: openssl rand -hex 32
docker compose up -d
docker compose exec marzban marzban-cli admin create --sudo
```

Access panel via SSH tunnel: `ssh -L 8000:localhost:8000 hetzner-vpn`  
Then open `http://localhost:8000`

### 2. Node on local machine

```bash
cd node
cp .env.example .env
# fill SERVICE_ADDRESS with your public IP or domain
docker compose up -d
```

### 3. Connect node to panel

1. Panel → **Settings → Nodes → Show Certificate** → copy cert
2. Panel → **Nodes → Add Node**:
   - Address: `<local public IP or domain>`
   - Port: `62050`
   - API Port: `62051`
   - Paste the certificate
3. Add inbounds for both Hetzner and local IPs

## Ports

| Port | Service | Where |
|------|---------|-------|
| `8000` | Panel UI | Hetzner (localhost only) |
| `8443` | VLESS+REALITY | both |
| `62050` | Node service | both |
| `62051` | Node API | both |

## Notes

- Panel is on Hetzner (stable public IP) — manages all users and node connections
- Local machine needs ports `8443`, `62050`, `62051` forwarded in your router. If you already run `vless-reality` locally, `8443` is done — add `62050`/`62051` for the node API
- Marzban uses SQLite by default — sufficient for personal use; MariaDB option is in the compose if needed
- Back up `panel/data/` (contains DB + Xray config)
