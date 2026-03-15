# Teleproxy

Telegram proxy bundle: **MTProto (Fake TLS)** + **SOCKS5** with shared user management, Zabbix monitoring, and a single CLI tool.

## What's inside

| Service | Image | Port | Protocol |
|---------|-------|------|----------|
| [telemt](https://github.com/telemt/telemt) | `whn0thacked/telemt-docker` | 8443 | MTProto Fake TLS |
| [3proxy](https://github.com/3proxy/3proxy) | `3proxy/3proxy` | 48731 | SOCKS5 with auth |

Both services share the same user credentials. Adding a user via `telemt-ctl` registers them in both MTProto and SOCKS5 simultaneously.

## Quick start

### 1. Clone and configure

```bash
git clone https://github.com/nordz0r/teleproxy.git /opt/teleproxy
cd /opt/teleproxy
cp data/config.toml.example data/config.toml
```

Edit `data/config.toml`:
- Set `public_host` to your server's domain
- Set `tls_domain` to a site for TLS fronting (e.g. `www.google.com`)

Edit `telemt-ctl`:
- Set `PUBLIC_HOST` to your server's domain

### 2. Set permissions

```bash
chown -R 65532:65532 data/
```

The telemt container runs as UID 65532 (nonroot) and needs write access to persist user changes via API.

### 3. Start

```bash
docker compose up -d
```

### 4. Add users

```bash
./telemt-ctl add alice
```

Output includes ready-to-use `tg://proxy` and `tg://socks` links for both protocols.

### 5. Symlink for convenience

```bash
ln -sf /opt/teleproxy/telemt-ctl /usr/local/bin/telemt-ctl
```

## telemt-ctl

Single CLI tool for managing users across both proxies.

### User management

```bash
telemt-ctl add <username>      # Add user, print tg:// links
telemt-ctl del <username>      # Remove user
telemt-ctl list                # List all users with links
telemt-ctl link <username>     # Show links for one user
telemt-ctl stats [username]    # Connection stats (all or one)
```

### Example output

```
────────────────────────────────────────
 alice
  mtproto  tg://proxy?server=proxy.example.com&port=8443&secret=ee...
  socks5   tg://socks?server=proxy.example.com&port=48731&user=alice&pass=...
────────────────────────────────────────
 bob
  mtproto  tg://proxy?server=proxy.example.com&port=8443&secret=ee...
  socks5   tg://socks?server=proxy.example.com&port=48731&user=bob&pass=...
────────────────────────────────────────
```

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PUBLIC_HOST` | `proxy.example.com` | Hostname used in generated links |
| `MTPROTO_PORT` | `8443` | MTProto port in links |
| `SOCKS5_PORT` | `48731` | SOCKS5 port in links |

Override per-call: `PUBLIC_HOST=other.host telemt-ctl list`

## Zabbix monitoring

### Install UserParameters

```bash
cp zabbix/telemt.conf /etc/zabbix/zabbix_agent2.d/
systemctl restart zabbix-agent2
```

### Available items

| Key | Type | Description |
|-----|------|-------------|
| `telemt.health` | Passive | 1 = healthy, 0 = down |
| `telemt.users.count` | Passive | Number of users |
| `telemt.conns.total` | Passive | Total active connections |
| `telemt.octets.total` | Passive | Total bytes (B) |
| `telemt.ips.total` | Passive | Total unique IPs |

### Per-user LLD

Discovery rule: `telemt.user.discovery`

Item prototypes:
- `telemt.user.conns[{#USERNAME}]`
- `telemt.user.octets[{#USERNAME}]`
- `telemt.user.ips[{#USERNAME}]`

Recommended tags: `component: telemt`, `mtproto_user: {#USERNAME}`

### Suggested triggers

| Severity | Expression |
|----------|------------|
| High | `last(/host/telemt.health)=0` |
| Warning | `max(/host/telemt.conns.total,10m)=0` |

## Architecture

```
/opt/teleproxy/
├── docker-compose.yml
├── telemt-ctl              # Management script
├── data/
│   ├── config.toml         # Telemt config (managed by API)
│   ├── 3proxy.cfg          # SOCKS5 config
│   └── socks5_passwd       # SOCKS5 users (auto-managed)
└── zabbix/
    └── telemt.conf          # Zabbix Agent 2 UserParameters
```

### How user management works

`telemt-ctl add` does two things atomically:
1. Calls telemt API (`POST /v1/users`) — adds user to MTProto and persists to `config.toml`
2. Appends `user:CL:secret` to `socks5_passwd` for 3proxy

No container restart needed. Telemt API handles hot-reload, 3proxy reads the passwd file on each auth attempt.

### Networking

- **telemt** runs in bridge mode with port mapping
- **socks5** runs in `network_mode: host` — required for SOCKS5 UDP ASSOCIATE to work (3proxy needs to bind arbitrary UDP ports for relay)

## Important: voice calls

Telegram's built-in proxy settings (both MTProto and SOCKS5) **do not support voice/video calls**. This is a [known limitation](https://github.com/telegramdesktop/tdesktop/issues/29665) of the Telegram client — it does not implement SOCKS5 UDP ASSOCIATE for call traffic.

**What works:**
- Messages, media, stickers, file transfers — fully work through both proxies
- Voice/video calls — require a system-wide VPN or external proxifier app (Nekobox, Shadowrocket, v2rayN)

## Requirements

- Docker + Docker Compose v2
- `jq` and `xxd` on the host (for `telemt-ctl`)
- Zabbix Agent 2 (optional, for monitoring)

## Ports

| Port | Protocol | Service |
|------|----------|---------|
| 8443 | TCP | MTProto Fake TLS |
| 48731 | TCP+UDP | SOCKS5 |
| 9090 | TCP (localhost) | Prometheus metrics |
| 9091 | TCP (localhost) | Telemt management API |

## Links

- [telemt](https://github.com/telemt/telemt) — MTProto proxy for Telegram (Rust + Tokio)
- [telemt-docker](https://github.com/An0nX/telemt-docker) — Docker image for telemt
- [3proxy](https://github.com/3proxy/3proxy) — Tiny free proxy server
- [Telegram calls issue](https://github.com/telegramdesktop/tdesktop/issues/29665) — Calls don't work through built-in proxy
