# WireGuard через reverse SSH

**Русский** · [English](README.en.md)

`Client:localhost:51820/udp ↔ reverse SSH ↔ Server:WireGuard UDP-порт`

Один контейнер на каждой стороне. **Server подключается по SSH к Client.**
Нужны Linux, Docker Compose и доступный с Server SSH-порт Client.

**Производительность.** WireGuard обычно передаёт пакеты по UDP. Здесь они идут через одно TCP-соединение SSH, поэтому пропускная способность может быть ощутимо ниже, чем при прямом подключении по UDP. При потере пакета TCP задерживает доставку следующих данных до повторной передачи.

## 1. Настройки

В папках `client/` и `server/` создайте рабочий `.env` из примера:

```sh
cp .env-example .env
```

В `client/.env`:

- `CLIENT_SSH_BIND`, `CLIENT_SSH_PORT` — где открыть SSH, по умолчанию `0.0.0.0:2222`.
- `LOCAL_WG_BIND`, `LOCAL_WG_PORT` — где открыть UDP, по умолчанию `127.0.0.1:51820`.

В `server/.env`:

- `CLIENT_SSH_HOST` — IP/DNS Client вместо `CHANGE_ME`.
- `CLIENT_SSH_PORT` — доступный с Server SSH-порт Client.
- `WG_TARGET_HOST`, `WG_TARGET_PORT` — WireGuard на Server, по умолчанию
  `host.docker.internal:51820`. Он должен быть доступен из Docker-сети.

`BRIDGE_TCP_PORT` должен совпадать на обеих сторонах. Остальное можно оставить как есть.

## 2. SSH-ключи

Один раз в корне проекта. Если ключи уже есть, пропустите этот шаг.

```sh
set -eu
umask 077
test ! -e client/secrets && test ! -e server/secrets
mkdir -p client/secrets server/secrets
ssh-keygen -q -t ed25519 -N '' -f server/secrets/id_ed25519
ssh-keygen -q -t ed25519 -N '' -f client/secrets/ssh_host_ed25519_key
cp server/secrets/id_ed25519.pub client/secrets/authorized_keys
awk '{print "reverse-wg-a " $1 " " $2}' client/secrets/ssh_host_ed25519_key.pub > server/secrets/known_hosts
```

## 3. Запуск

Перенесите целиком `client/` на Client, `server/` — на Server, включая `secrets/`
и скрытые файлы (`.env`, `.dockerignore`, `.gitignore`). Каждая папка самостоятельна;
файлы из корня проекта для запуска не нужны.

В папке `client/` на Client и `server/` на Server выполните:

```sh
docker compose up -d --build --remove-orphans
```

Временное состояние wg-bridge хранится в `tmpfs` и очищается при перезапуске,
чтобы старый PID не блокировал восстановление туннеля.

## 4. Пример использования WireGuard на Client

В конфиге WireGuard на хосте Client укажите:

```ini
Endpoint = 127.0.0.1:51820
PersistentKeepalive = 25
```

Используйте адрес и порт из `LOCAL_WG_BIND` / `LOCAL_WG_PORT`;
для привязки `0.0.0.0` укажите в Endpoint конкретный доступный IP Client.
Собственный `ListenPort` WireGuard должен отличаться от `LOCAL_WG_PORT`.
При full tunnel сохраните маршрут SSH к Server вне VPN.

## Проверка

```sh
docker compose logs --tail=50 transport
```

В своём WireGuard-клиенте проверьте свежий handshake и ping VPN-адреса Server.
Сборка и сквозное соединение в среде подготовки не проверялись: Docker отсутствует.
