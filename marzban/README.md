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

1. Open V2Box → bottom bar → **Settings** (gear icon)
2. Tap **Routing**
3. Tap **Routing Mode** → select **Custom**
4. Tap **Edit routing rules** → replace the content with:

```json
{
  "domainStrategy": "IPIfNonMatch",
  "rules": [
    {
      "type": "field",
      "ip": ["geoip:private"],
      "outboundTag": "direct"
    },
    {
      "type": "field",
      "ip": ["geoip:ru"],
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

5. Tap **Save** → reconnect

> `domainStrategy: IPIfNonMatch` means V2Box resolves a domain to IP first if no domain rule matches, then checks IP rules. This catches Russian IPs that don't have a `.ru` domain.

---

### Shadowrocket

**Step 1 — switch to rule-based routing**

On the home screen, tap the routing mode label (shows `PROXY`, `CONFIG`, or `DIRECT`) → select **CONFIG**.

**Step 2 — open the active config**

Bottom bar → **Config** tab → tap the active `.conf` file (has a blue checkmark) → **Edit**

**Step 3 — add rules**

Find the `[Rule]` section. Add these lines **at the top** of that section, before any existing rules and before `FINAL`:

```
# Russian IPs — direct
GEOIP,RU,DIRECT

# Russian domains — direct
RULE-SET,https://raw.githubusercontent.com/nicksyoshe/FreedomListVPN/main/shadowrocket-whitelist-ru.conf,DIRECT

# Everything else — through VPN
FINAL,PROXY
```

Tap **Save** in the top right.

**Step 4 — reload config**

Back on the **Config** tab → tap your config → **Use Config**. Toggle VPN off and on.

> If you don't have a `[Rule]` section at all, add it manually. A minimal config looks like:
> ```
> [General]
> [Rule]
> GEOIP,RU,DIRECT
> FINAL,PROXY
> ```

---

### What `geosite:ru` / the RULE-SET covers

Yandex, VK, Mail.ru, Sber, Tinkoff, Gosuslugi, Ozon, Wildberries, and thousands of other `.ru` domains. Updated automatically when clients update geo databases (V2Box) or when Shadowrocket refreshes the remote RULE-SET.

> **A site still routes through VPN?** It may have a non-`.ru` domain or a CDN that serves non-RU IPs. Add it manually:
> - V2Box: add a `"domain": ["domain:example.com"]` rule → `"outboundTag": "direct"`
> - Shadowrocket: **Config → Rules → +** → Type: `DOMAIN-SUFFIX`, Value: `example.com`, Policy: `DIRECT`

---

## Notes

- Ports bound to `${SERVER_IP}` (not `0.0.0.0`) to avoid conflict with Tailscale which also binds to port 443 on its own interface
- Panel uses a CA-signed cert (not bare self-signed) — Marzban rejects bare self-signed certs
- Local node needs ports `8443`, `62050`, `62051` forwarded in your router; DuckDNS handles dynamic IP
- Back up `panel/data/marzban/` — contains DB, Xray config, and certs
