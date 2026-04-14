[EN](./README.md) | [RU](./README.ru.md)

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

**SSL setup** (required — without this the panel binds to 127.0.0.1 only):

```bash
mkdir -p data/marzban/certs
# Create a local CA
openssl genrsa -out data/marzban/certs/ca.key 4096
openssl req -new -x509 -days 3650 -key data/marzban/certs/ca.key \
  -out data/marzban/certs/ca.pem -subj '/CN=marzban-ca'
# Sign a server cert with it
openssl genrsa -out data/marzban/certs/key.pem 4096
openssl req -new -key data/marzban/certs/key.pem \
  -out data/marzban/certs/server.csr -subj '/CN=marzban-panel'
openssl x509 -req -days 3650 -in data/marzban/certs/server.csr \
  -CA data/marzban/certs/ca.pem -CAkey data/marzban/certs/ca.key \
  -CAcreateserial -out data/marzban/certs/cert.pem
```

**Start:**

```bash
cp .env.example .env
# fill in SERVER_IP and SECRET_KEY (openssl rand -hex 32)
docker compose up -d
```

**Create admin** (non-interactive):

```bash
MARZBAN_ADMIN_PASSWORD='yourpassword' \
  docker exec -e MARZBAN_ADMIN_PASSWORD='yourpassword' marzban \
  marzban-cli admin create -u admin --sudo
```

**Access panel** via SSH tunnel:

```bash
ssh -L 8000:localhost:8000 user@server
```

Then open **`https://localhost:8000/dashboard/`** (accept the cert warning — self-signed CA).

### 2. Configure VLESS+REALITY inbound via API

```bash
# Get token
TOKEN=$(curl -sk -X POST https://localhost:8000/api/admin/token \
  -d 'username=admin&password=yourpassword' | jq -r .access_token)

# Generate Reality keypair
docker exec marzban xray x25519

# Push inbound config (replace privateKey with generated value)
curl -sk -X PUT https://localhost:8000/api/core/config \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d @xray_config.json
```

### 3. Node on local machine

```bash
cd node
cp .env.example .env
# fill SERVICE_ADDRESS with your DuckDNS domain
docker compose up -d
```

In panel → **Nodes → Add Node**: address = DuckDNS domain, ports 62050/62051, paste panel cert.

## Ports

| Port | Purpose | Bound to |
|------|---------|----------|
| `8000` | Panel UI (HTTPS) | `127.0.0.1` (SSH tunnel only) |
| `443` | VLESS+REALITY inbound | public IP only |
| `62050` | Node service port | public IP only |
| `62051` | Node API port | public IP only |

## Clients

### Getting your connection link

Open the panel → click your username → copy the `vless://` link or scan the QR code.

> The subscription URL (`/sub/...`) only works while the SSH tunnel is active. For permanent access share the `vless://` link directly.

---

### V2Box (iOS / macOS)

1. Tap **+** → **Import from clipboard** (paste the `vless://` link)
2. Tap the config → **Edit** → enable **"Mux"** if connection feels slow
3. For split tunneling → see below

---

### Shadowrocket (iOS)

1. Tap **+** → paste the `vless://` link → **Save**
2. Tap the server row to set it as active
3. For split tunneling → see below

---

## Split tunneling — bypass VPN for Russian sites

Route Russian IPs and domains directly so banking apps, Yandex, VK, etc. work normally and don't see a foreign IP.

### V2Box

Go to **Settings → Routing → Rules** and add two rules **before** any existing proxy rules:

| Field | Value |
|-------|-------|
| Type | `geoip` |
| Value | `ru` |
| Outbound | `direct` |

| Field | Value |
|-------|-------|
| Type | `geosite` |
| Value | `ru` |
| Outbound | `direct` |

Or switch to **Custom** routing and paste:

```json
{
  "domainStrategy": "IPIfNonMatch",
  "rules": [
    {
      "type": "field",
      "ip": ["geoip:ru", "geoip:private"],
      "outboundTag": "direct"
    },
    {
      "type": "field",
      "domain": ["geosite:ru"],
      "outboundTag": "direct"
    }
  ]
}
```

### Shadowrocket

Go to **Config → Edit configuration** and add these lines in the `[Rule]` section, **before** `FINAL`:

```
GEOIP,RU,DIRECT
RULE-SET,https://raw.githubusercontent.com/nicksyoshe/FreedomListVPN/main/shadowrocket-whitelist-ru.conf,DIRECT
FINAL,PROXY
```

Or use the built-in approach: **Global Routing → Rule** mode, then in **Add Rule** → **GEOIP** → `RU` → `DIRECT`.

### What `geosite:ru` covers

Yandex, VK, Mail.ru, Sber, Tinkoff, Gosuslugi, Ozon, Wildberries, and ~3000 other `.ru` domains. Updated automatically when clients pull fresh geo databases.

> If a Russian site still doesn't work through direct, it may block non-RU IPs at the CDN level — add its domain manually as a `domain` rule → `direct`.

---

## Notes

- Ports bound to `${SERVER_IP}` (not `0.0.0.0`) to avoid conflict with Tailscale which also binds to port 443 on its own interface
- Panel uses a CA-signed cert (not bare self-signed) — Marzban rejects bare self-signed certs
- Local node needs ports `8443`, `62050`, `62051` forwarded in your router; DuckDNS handles dynamic IP
- Back up `panel/data/marzban/` — contains DB, Xray config, and certs
