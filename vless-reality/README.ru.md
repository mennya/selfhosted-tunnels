[EN](./README.md) | [RU](./README.ru.md)

# VLESS + REALITY через 3X-UI

Самостоятельно размещённый VPN на базе [3X-UI](https://github.com/MHSanaei/3x-ui) (панель Xray) с транспортом VLESS+REALITY и DuckDNS для динамического DNS.

REALITY имитирует настоящее TLS-рукопожатие с реальным сайтом-назначением (например, `www.microsoft.com`), делая трафик неотличимым от обычного HTTPS.

## Требования

- VPS с публичным IP
- Docker + Docker Compose
- Аккаунт на [DuckDNS](https://www.duckdns.org)

## Установка

```bash
cp .env.example .env
# заполнить TZ, DUCKDNS_SUBDOMAIN, DUCKDNS_TOKEN
docker compose up -d
```

**Первоначальная настройка панели:**

1. Открыть `http://localhost:2053` (если удалённо — SSH-туннель: `ssh -L 2053:localhost:2053 user@server`)
2. Войти: `admin` / `admin` — **сменить немедленно** в Settings → User
3. Inbounds → Add Inbound:
   - Protocol: `vless`
   - Port: `8443`
   - Security: `reality`
   - Dest (uTLS): `www.microsoft.com:443`
   - SNI: `www.microsoft.com`
   - Сгенерировать ключи, сохранить
4. Скопировать ссылку / QR-код и передать клиенту

## Порты

| Порт | Назначение |
|------|---------|
| `2053` | Панель 3X-UI (привязана к `127.0.0.1`) |
| `8443` | VLESS+REALITY inbound (публичный) |

## Клиенты

- **iOS/macOS**: [Streisand](https://apps.apple.com/app/streisand/id6450534064)
- **Android**: [v2rayNG](https://github.com/2dust/v2rayNG)
- **Windows/Linux**: [Hiddify](https://github.com/hiddify/hiddify-app)

## Примечания

- Панель привязана только к `127.0.0.1`; при удалённом управлении использовать SSH-туннель
- Дайджесты образов зафиксированы — это предотвращает неожиданные обновления, ломающие конфигурацию
- 3X-UI хранит всю конфигурацию в `./data/3x-ui` — сделайте резервную копию после первоначальной настройки
