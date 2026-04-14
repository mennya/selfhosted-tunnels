[EN](./README.md) | [RU](./README.ru.md)

# Marzban — Многоузловая VPN-панель

Схема основного/резервного узлов на базе [Marzban](https://github.com/Gozargah/Marzban) + [Marzban Node](https://github.com/Gozargah/Marzban-node). Пользователи получают inbound-ы на обоих серверах в одной конфигурации — клиенты переключаются автоматически.

## Архитектура

```
Hetzner (panel/)          ← основной inbound + панель управления
    │ TLS node API
Локальная машина (node/)  ← резервный inbound, автопереключение
```

## Порядок развёртывания

### 1. Панель на Hetzner

**Настройка SSL** (обязательно — без этого панель привязывается только к 127.0.0.1):

```bash
mkdir -p data/marzban/certs
# Создать локальный CA
openssl genrsa -out data/marzban/certs/ca.key 4096
openssl req -new -x509 -days 3650 -key data/marzban/certs/ca.key \
  -out data/marzban/certs/ca.pem -subj '/CN=marzban-ca'
# Подписать серверный сертификат через CA
openssl genrsa -out data/marzban/certs/key.pem 4096
openssl req -new -key data/marzban/certs/key.pem \
  -out data/marzban/certs/server.csr -subj '/CN=marzban-panel'
openssl x509 -req -days 3650 -in data/marzban/certs/server.csr \
  -CA data/marzban/certs/ca.pem -CAkey data/marzban/certs/ca.key \
  -CAcreateserial -out data/marzban/certs/cert.pem
```

**Запуск:**

```bash
cp .env.example .env
# заполнить SERVER_IP и SECRET_KEY (openssl rand -hex 32)
docker compose up -d
```

**Создание администратора** (неинтерактивно):

```bash
MARZBAN_ADMIN_PASSWORD='yourpassword' \
  docker exec -e MARZBAN_ADMIN_PASSWORD='yourpassword' marzban \
  marzban-cli admin create -u admin --sudo
```

**Доступ к панели** через SSH-туннель:

```bash
ssh -L 8000:localhost:8000 user@server
```

Открыть **`https://localhost:8000/dashboard/`** (принять предупреждение о сертификате — самоподписанный CA).

### 2. Настройка VLESS+REALITY inbound через API

```bash
# Получить токен
TOKEN=$(curl -sk -X POST https://localhost:8000/api/admin/token \
  -d 'username=admin&password=yourpassword' | jq -r .access_token)

# Сгенерировать ключевую пару Reality
docker exec marzban xray x25519

# Отправить конфигурацию inbound (заменить privateKey на сгенерированный)
curl -sk -X PUT https://localhost:8000/api/core/config \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d @xray_config.json
```

### 3. Node на локальной машине

```bash
cd node
cp .env.example .env
# заполнить SERVICE_ADDRESS — DuckDNS-домен локальной машины
docker compose up -d
```

В панели → **Nodes → Add Node**: адрес = DuckDNS-домен, порты 62050/62051, вставить сертификат панели.

## Порты

| Порт | Назначение | Привязан к |
|------|---------|----------|
| `8000` | Панель (HTTPS) | `127.0.0.1` (только SSH-туннель) |
| `443` | VLESS+REALITY inbound | только публичный IP |
| `62050` | Порт сервиса node | только публичный IP |
| `62051` | API порт node | только публичный IP |

## Клиенты

### Получение ссылки для подключения

Открыть панель → нажать на имя пользователя → скопировать ссылку `vless://` или отсканировать QR-код.

> Ссылка-подписка (`/sub/...`) работает только пока активен SSH-туннель. Для постоянного доступа передавать ссылку `vless://` напрямую.

---

### V2Box (iOS / macOS)

1. Нажать **+** → **Import from clipboard** (вставить ссылку `vless://`)
2. Нажать на конфигурацию → **Edit** → включить **"Mux"** если соединение медленное
3. Раздельное туннелирование → см. ниже

---

### Shadowrocket (iOS)

1. Нажать **+** → вставить ссылку `vless://` → **Save**
2. Нажать на строку сервера для активации
3. Раздельное туннелирование → см. ниже

---

## Раздельное туннелирование — российские сайты без VPN

Российские IP и домены идут напрямую — банковские приложения, Яндекс, ВКонтакте и т.д. работают нормально и видят российский IP.

### V2Box

1. Открыть V2Box → нижняя панель → **Settings** (иконка шестерёнки)
2. Нажать **Routing**
3. Нажать **Routing Mode** → выбрать **Custom**
4. Нажать **Edit routing rules** → заменить содержимое на:

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

5. Нажать **Save** → переподключиться

> `domainStrategy: IPIfNonMatch` — если домен не совпадает ни с одним правилом, V2Box резолвит его в IP и проверяет IP-правила. Это позволяет поймать российские IP у сайтов без домена `.ru`.

---

### Shadowrocket

**Шаг 1 — включить маршрутизацию по правилам**

На главном экране нажать на метку режима (показывает `PROXY`, `CONFIG` или `DIRECT`) → выбрать **CONFIG**.

**Шаг 2 — открыть активный конфиг**

Нижняя панель → вкладка **Config** → нажать на активный файл `.conf` (синяя галочка) → **Edit**

**Шаг 3 — добавить правила**

Найти секцию `[Rule]`. Добавить эти строки **в самое начало** секции, до всех существующих правил и до `FINAL`:

```
# Российские IP — напрямую
GEOIP,RU,DIRECT

# Российские домены — напрямую
RULE-SET,https://raw.githubusercontent.com/nicksyoshe/FreedomListVPN/main/shadowrocket-whitelist-ru.conf,DIRECT

# Всё остальное — через VPN
FINAL,PROXY
```

Нажать **Save** в правом верхнем углу.

**Шаг 4 — применить конфиг**

Вернуться на вкладку **Config** → нажать на конфиг → **Use Config**. Выключить и снова включить VPN.

> Если секции `[Rule]` нет вообще — добавить вручную. Минимальный конфиг выглядит так:
> ```
> [General]
> [Rule]
> GEOIP,RU,DIRECT
> FINAL,PROXY
> ```

---

### Что входит в `geosite:ru` / RULE-SET

Яндекс, ВКонтакте, Mail.ru, Сбер, Тинькофф, Госуслуги, Ozon, Wildberries и тысячи других `.ru`-доменов. Обновляется автоматически — в V2Box при обновлении геобаз, в Shadowrocket при обновлении удалённого RULE-SET.

> **Сайт всё равно идёт через VPN?** Возможно, у него не `.ru`-домен или CDN отдаёт не российские IP. Добавить вручную:
> - V2Box: правило `"domain": ["domain:example.com"]` → `"outboundTag": "direct"`
> - Shadowrocket: **Config → Rules → +** → Type: `DOMAIN-SUFFIX`, Value: `example.com`, Policy: `DIRECT`

---

## Примечания

- Порты привязаны к `${SERVER_IP}` (не `0.0.0.0`), чтобы избежать конфликта с Tailscale, который тоже занимает порт 443 на своём интерфейсе
- Панель использует сертификат, подписанный локальным CA (не самоподписанный напрямую) — Marzban отклоняет «голые» самоподписанные сертификаты
- На локальной машине необходимо пробросить порты `8443`, `62050`, `62051` в роутере; DuckDNS обрабатывает динамический IP
- Резервная копия `panel/data/marzban/` — содержит БД, конфигурацию Xray и сертификаты
