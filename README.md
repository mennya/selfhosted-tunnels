# selfhosted-tunnels

Docker Compose setups for self-hosted VPN and proxy services. Each directory is a standalone, deployable stack.

## Stacks

| Directory | Stack | Use case |
|-----------|-------|----------|
| [`vless-reality/`](./vless-reality/) | 3X-UI + VLESS+REALITY + DuckDNS | Censorship-resistant VPN, single node |
| [`marzban/`](./marzban/) | Marzban panel + node | Multi-node VPN, primary/secondary failover |

## Structure

Each stack contains:
- `docker-compose.yml` — ready to deploy
- `.env.example` — all required variables with descriptions
- `README.md` — setup steps, port map, client recommendations

## Usage

```bash
cd <stack>
cp .env.example .env
# edit .env
docker compose up -d
```
