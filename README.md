# WireGuard over reverse SSH

[Русский](README.ru.md) · **English**

`Client:localhost:51820/udp ↔ reverse SSH ↔ Server:WireGuard UDP port`

Each side runs one container. **Server initiates the SSH connection to Client.**
You need Linux, Docker Compose, and an SSH port on Client that Server can reach.

**Performance.** WireGuard normally sends packets over UDP. This tunnel carries them over a single SSH TCP connection, so throughput can be noticeably lower than with a direct UDP connection. When a packet is lost, TCP holds up delivery of subsequent data until retransmission.

## 1. Configuration

Run this command in both `client/` and `server/` to create each side’s `.env` file.

```sh
cp .env-example .env
```

In `client/.env`, configure the listening addresses and ports.

- `CLIENT_SSH_BIND` and `CLIENT_SSH_PORT` control where SSH listens. The default is `0.0.0.0:2222`.
- `LOCAL_WG_BIND` and `LOCAL_WG_PORT` control where the tunnel accepts UDP packets. The default is `127.0.0.1:51820`.

In `server/.env`, set the tunnel destinations.

- `CLIENT_SSH_HOST` — replace `CHANGE_ME` with Client’s IP address or DNS name.
- `CLIENT_SSH_PORT` — the SSH port on Client that Server can reach.
- `WG_TARGET_HOST` and `WG_TARGET_PORT` — the WireGuard endpoint on Server. The default is `host.docker.internal:51820`. The endpoint must be reachable from the Docker network.

Use the same `BRIDGE_TCP_PORT` on both sides. You can leave the other settings at their defaults.

## 2. SSH keys

Run these commands once from the project root. Skip this step if the keys already exist.

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

## 3. Start the tunnel

Copy the entire `client/` directory to Client and `server/` to Server. Include `secrets/` and the hidden files (`.env`, `.dockerignore`, and `.gitignore`). Each directory contains everything that side needs to run.

Run this command in `client/` on Client and in `server/` on Server.

```sh
docker compose up -d --build --remove-orphans
```

wg-bridge stores temporary state in `tmpfs`. Restarting the container clears it so a stale PID cannot prevent the tunnel from reconnecting.

## 4. Configure WireGuard on Client

Use the following settings in the WireGuard peer configuration on the Client host.

```ini
Endpoint = 127.0.0.1:51820
PersistentKeepalive = 25
```

Use the address and port from `LOCAL_WG_BIND` and `LOCAL_WG_PORT`. If the tunnel binds to `0.0.0.0`, set `Endpoint` to a specific, reachable IP address on Client and the configured port.
WireGuard’s own `ListenPort` must differ from `LOCAL_WG_PORT`.
For a full-tunnel VPN, keep the route for SSH traffic to Server outside the VPN.

## Verify the connection

```sh
docker compose logs --tail=50 transport
```

Check for a recent handshake in your WireGuard client and ping Server’s VPN address.
The build and end-to-end connection were untested in the original setup environment because Docker was unavailable there.
