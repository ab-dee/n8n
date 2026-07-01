# n8n Self-Hosted

Docker Compose deployment of [n8n](https://n8n.io) workflow automation on a single Server.

---

## Deployment Profiles

| Profile | Compose File | Description |
|---|---|---|
| **Production** | `compose.yaml` | Caddy reverse proxy, automatic TLS, PostgreSQL, persistent volumes |
| **VPN** | `compose.vpn.yaml` | Direct port access over VPN tunnel, PostgreSQL, persistent volumes |
| **VPN (SQLite)** | `compose.vpn.sqlite.yaml` | Direct port access over VPN tunnel, SQLite, persistent volume; no database container to manage |
| **Ephemeral** | `compose.ephemeral.yaml` | Single container, SQLite, no volumes; data is not retained across restarts |

### Production

Requires a DNS A record pointing to the VPS and ports 80/443 open. Caddy provisions and renews TLS certificates automatically.

```bash
cd workspace
docker compose up -d
```

### VPN

Exposes n8n on port 5678. Port 5678 must be restricted to VPN-sourced traffic only — do not expose it to the public internet. Set `WEBHOOK_URL=http://<VPS-IP>:5678` and `N8N_PROXY_HOPS=0` in `.env`.

```bash
cd workspace
docker compose -f compose.vpn.yaml up -d
```

### VPN (SQLite)

Same access pattern as VPN, but uses SQLite instead of PostgreSQL — no separate database container to run or back up. Data persists in the `n8n_data` volume across restarts and redeployments. Port 5678 must be restricted to VPN-sourced traffic only. Set `WEBHOOK_URL=http://<VPS-IP>:5678` and `N8N_PROXY_HOPS=0` in `.env`. Only `N8N_ENCRYPTION_KEY` (and optionally `GENERIC_TIMEZONE`/`TZ`) are required — the `DB_POSTGRESDB_*` variables are not used by this profile.

This profile sets `N8N_SECURE_COOKIE=false`. It's accessed over plain HTTP (e.g. via a Tailscale IP), and browsers won't store/send a `Secure` cookie on a non-HTTPS origin — Tailscale's own tunnel encryption doesn't change what the browser sees, so without this the UI shows a secure-cookie warning and sessions won't persist.

```bash
cd workspace
docker compose -f compose.vpn.sqlite.yaml up -d
```

### Ephemeral

Runs a single n8n container with SQLite and no mounted volumes. All data is lost when the stack is stopped. No `.env` file is required.

```bash
cd workspace
docker compose -f compose.ephemeral.yaml up
docker compose -f compose.ephemeral.yaml down
```

---

## Production Pre-Deploy Checklist

1. Point a DNS A record for your domain to the Server IP.
2. Open ports **80** and **443** on the Servers firewall.
3. Copy the environment template:
   ```bash
   cp workspace/.env.example workspace/.env
   ```
4. Generate an encryption key and set it as `N8N_ENCRYPTION_KEY` in `.env`:
   ```bash
   openssl rand -hex 32
   ```
   Back up this value to an external secrets manager. It cannot be recovered, and losing it renders all stored credentials unreadable.
5. Create the shared files directory with the correct ownership:
   ```bash
   mkdir -p workspace/local-files
   sudo chown -R 1000:1000 workspace/local-files
   ```
6. Start the stack:
   ```bash
   cd workspace
   docker compose up -d
   ```
7. Complete owner account setup at `https://yourdomain.com` before the instance is reachable by others.

---

## Updates

The n8n image is pinned to a specific version tag in each Compose file. To update:

```bash
cd workspace
docker compose pull
docker compose up -d
```

Review the [n8n changelog](https://github.com/n8n-io/n8n/releases) before updating. Update monthly or immediately when a security patch is released.

---

## Backup

| Item | Recommended Method |
|---|---|
| Workflows and credentials (PostgreSQL profiles) | Scheduled `pg_dump` to off-site storage |
| Workflows and credentials (SQLite profile) | Snapshot/copy of the `n8n_data` volume's `database.sqlite` file |
| `N8N_ENCRYPTION_KEY` | External secrets manager |
| `n8n_data` Docker volume | Snapshot alongside the database |

---

## Directory Layout

```
workspace/
├── compose.yaml              # Production profile
├── compose.vpn.yaml          # VPN profile
├── compose.vpn.sqlite.yaml   # VPN profile (SQLite)
├── compose.ephemeral.yaml    # Ephemeral profile
├── Caddyfile                 # Caddy configuration (production only)
├── .env.example              # Environment variable template
├── .gitignore
└── local-files/              # Host directory mounted for file nodes
```
