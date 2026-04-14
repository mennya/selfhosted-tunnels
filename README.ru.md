[EN](./README.md) | [RU](./README.ru.md)

# selfhosted-tunnels

Docker Compose конфигурации для самостоятельно размещённых VPN и прокси-сервисов. Каждая директория — отдельный, готовый к развёртыванию стек.

## Стеки

| Директория | Стек | Назначение |
|-----------|-------|----------|
| [`vless-reality/`](./vless-reality/README.ru.md) | 3X-UI + VLESS+REALITY + DuckDNS | VPN с защитой от цензуры, один узел |
| [`marzban/`](./marzban/README.ru.md) | Marzban panel + node | Многоузловой VPN, основной/резервный режим |

## Структура

Каждый стек содержит:
- `docker-compose.yml` — готово к запуску
- `.env.example` — все необходимые переменные с описанием
- `README.md` — шаги установки, таблица портов, рекомендации по клиентам

## Использование

```bash
cd <стек>
cp .env.example .env
# заполнить .env
docker compose up -d
```
