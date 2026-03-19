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
cp .env.example .env
cp data/config.toml.example data/config.toml
```

Edit `.env`:
- Set `TELEPROXY_DOMAIN` to your server's domain
- Optionally override `PUBLIC_HOST`, `TLS_DOMAIN`, `MTPROTO_PORT`, `SOCKS5_PORT`

> ⚠️ If you run Telegram through xray/sing-box (or another SNI-aware L7 router),
> using third-party `TLS_DOMAIN` like `www.google.com` can break MTProto:
> the router may route traffic by SNI to the original website instead of your telemt host.
> Use a domain you control that points to your telemt server.

Generate or refresh `data/config.toml` host fields from env:

```bash
./telemt-ctl config-sync-env
```

`telemt-ctl` reads `/opt/teleproxy/.env` by default, so the env file is the source of truth
for generated links and for the host-related values written into `data/config.toml`.

### FAQ: domain from env + certificates

**Q: I already have a domain in an environment variable. Can I reuse it?**  
Yes. Use the same domain value in both places:
- `TELEPROXY_DOMAIN` / `PUBLIC_HOST` (for generated links in `telemt-ctl`)
- `TLS_DOMAIN` in `data/config.toml` via `telemt-ctl config-sync-env`

Example:

```bash
export TELEPROXY_DOMAIN=proxy.example.com
./telemt-ctl config-sync-env
./telemt-ctl list
```

To override values in your current terminal or interactive shell,
export them manually or put them in `.env` / `~/.bashrc`:

```bash
export TELEPROXY_DOMAIN=proxy.example.com
export PUBLIC_HOST="$TELEPROXY_DOMAIN"
export MTPROTO_PORT=8443
export SOCKS5_PORT=48731
```

If you run `telemt-ctl` through a non-interactive SSH command like
`ssh root@host '/opt/teleproxy/telemt-ctl list'`, `~/.bashrc` may not be loaded.
`telemt-ctl` avoids that problem by loading `/opt/teleproxy/.env` directly.
If you keep env only in `~/.bashrc`, pass the vars inline or source the file explicitly.

After `./telemt-ctl config-sync-env`, `data/config.toml` should contain:

```toml
[general.links]
public_host = "proxy.example.com"

[censorship]
tls_domain = "proxy.example.com"
```

**Q: Do I need a valid TLS certificate for `tls_domain`?**  
For telemt MTProto Fake TLS itself, a public CA certificate is generally **not required**.  
But if you place a real TLS terminator in front (Nginx/Caddy/CDN/reverse-proxy), then that front layer needs a valid certificate as usual.

### 2. Set permissions

```bash
chown -R 65532:65532 data/
```

The telemt container runs as UID 65532 (nonroot) and needs write access to persist user changes via API.

### 3. Start

```bash
./telemt-ctl config-sync-env
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
| `TELEPROXY_DOMAIN` | `proxy.example.com` | Shared default for links and TLS SNI |
| `PUBLIC_HOST` | `TELEPROXY_DOMAIN` | Hostname used in generated links |
| `TLS_DOMAIN` | `TELEPROXY_DOMAIN` | TLS SNI written into `data/config.toml` |
| `MTPROTO_PORT` | `8443` | MTProto port in links and config |
| `SOCKS5_PORT` | `48731` | SOCKS5 port in links |
| `TELEPROXY_ENV_FILE` | `/opt/teleproxy/.env` | Path to env file loaded by `telemt-ctl` |

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

## Cluster mode (multi-server)

Teleproxy supports automatic replication across multiple servers. When you add or remove a user on any node, the change is propagated to all peers via SSH.

### Setup

1. **Configure SSH keys** between servers (root access):

```bash
# On each server, ensure passwordless SSH to all peers
ssh-copy-id root@node2.example.com
```

2. **Create peers config** on each server:

```bash
# On node1 — list all OTHER nodes
echo "node2.example.com" > /opt/teleproxy/data/peers.conf

# On node2 — list all OTHER nodes
echo "node1.example.com" > /opt/teleproxy/data/peers.conf
```

3. **Point DNS** to all nodes (round-robin A records):

```
tg.example.com  A  1.2.3.4    # node1
tg.example.com  A  5.6.7.8    # node2
```

### Usage

```bash
# Add user — automatically replicated to all peers
telemt-ctl add alice

# Remove user — automatically replicated
telemt-ctl del alice

# Full sync — push local state to all peers (add missing, remove extra)
telemt-ctl sync

# Check peer health
telemt-ctl peers

# Local-only operation (no replication, used internally)
telemt-ctl --local add alice
```

### How it works

- `add`/`del` execute locally first, then SSH to each peer and run `telemt-ctl --local` there
- `sync` treats the local server as the source of truth: adds missing users on peers, removes extra ones
- Users are added with the same secret on all nodes, so `tg://` links work regardless of which server handles the connection
- If a peer is unreachable during `add`/`del`, the operation continues — run `sync` later to fix drift

## Architecture

```
/opt/teleproxy/
├── docker-compose.yml
├── telemt-ctl              # Management script
├── data/
│   ├── config.toml         # Telemt config (managed by API)
│   ├── 3proxy.cfg          # SOCKS5 config
│   ├── socks5_passwd       # SOCKS5 users (auto-managed)
│   └── peers.conf          # Cluster peers (optional)
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
