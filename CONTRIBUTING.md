# Contributing to Risus CLI

## Prerequisites

- Python 3.12+
- One container runtime:
  - **Docker**: Docker Engine 24+ with the Compose plugin (`docker compose`)
  - **Podman**: Podman 4.7+ (`podman-compose` is installed via `.[dev]` below)

## Setup

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -e ".[dev]"
```

This installs all development tools including `podman-compose`, `ruff`, and `pytest`.

### Environment Variables

The next sections specify required environment variables. The
[envrc.example](./envrc.example) file can be copied to `.envrc` and
[direnv](https://direnv.net/) can automatically apply the file.

> [!CAUTION]
> Never commit `.envrc` to git. It contains secrets.

```bash
cp env.example .envrc

# edit .envrc
```

### Rootless Podman (optional)

For rootless Podman, expose the socket so `pytest-docker` can reach it:

```bash
systemctl --user start podman.socket
export DOCKER_HOST=unix:///run/user/$UID/podman/podman.sock
```

---

## Bring Up the Stack

The server requires a `RISUS_TOKEN` environment variable. Clients must supply the same token to connect.

```bash
export RISUS_TOKEN=your-secret-token-here   # min 16 chars
```

**Docker:**
```bash
docker compose up -d --build
```

**Podman:**
```bash
PATH=$PWD/.venv/bin:$PATH CONTAINER_ENGINE=podman podman-compose up -d --build
```

Both commands build the server image and start `db` (Postgres 16) and `server` (FastAPI on port 8765). Verify:

```bash
curl http://localhost:8765/healthz
# → {"ok":true}
```

---

## Run the CLI

```bash
python risus.py
```

You'll be prompted for server address, display name, and session token. Supply the same `RISUS_TOKEN` value set when starting the stack.

Pass values directly to skip prompts:

```bash
python risus.py localhost:8765 MyName --token your-secret-token-here
```

Token is saved to `risus.cfg` after a successful connection and reused on the next launch.

### Connecting

- Name must be unique per session. If a name is already connected the server closes your WebSocket with code `4409 "name in use"`. Choose a different name or wait ~30 s for the prior session's keepalive to expire.

### Reconnecting

- On reconnect, you start fresh. Any lock you held before the drop is released server-side; you must re-acquire it explicitly.

### CLI UX Limitation

> Remote changes appear after your next keypress, not mid-prompt.

The menu uses blocking `input()` to preserve the original single-player UX. While you're sitting at a prompt, changes from other clients accumulate in a queue and are rendered the next time the menu redraws (after you press Enter). This is intentional.

### Spectating

Any connected client receives all state/presence/lock broadcasts. To spectate, just connect and never lock or edit anything.

---

## Run Tests

### Unit tests (no containers required)

**Docker:**
```bash
pytest tests/unit -q
```

**Podman:**
```bash
pytest tests/unit -q
```

Unit tests are identical for both runtimes — no container dependency.

### E2E tests (require running containers)

**Docker:**
```bash
CONTAINER_ENGINE=docker RISUS_TOKEN=test-token-for-e2e pytest tests/e2e -m e2e -q
```

**Podman:**
```bash
PATH=$PWD/.venv/bin:$PATH CONTAINER_ENGINE=podman RISUS_TOKEN=test-token-for-e2e pytest tests/e2e -m e2e -q
```

E2E tests spin up and tear down the full stack automatically using the project's `docker-compose.yml`. `RISUS_TOKEN` is required — the server rejects connections without it.

For rootless Podman, also export:
```bash
export DOCKER_HOST=unix:///run/user/$UID/podman/podman.sock
```

---

## Reset State

Destroy all containers and volumes (wipes the database):

**Docker:**
```bash
docker compose down -v
```

**Podman:**
```bash
PATH=$PWD/.venv/bin:$PATH podman-compose down -v
```

> **Note:** Schema changes require `down -v` because Postgres initialises the schema on first boot from `server/schema.sql`. A running volume will not re-initialise.

---

## Production Deployment (Hetzner)

This guide assumes a Hetzner Cloud VM is already provisioned and SSH-accessible.

### 1. DNS — Hetzner KonsoleH

1. Log in to [konsoleh.hetzner.com](https://konsoleh.hetzner.com)
2. Navigate to **Settings → DNS-Management → Open DNS Panel**
3. Add an A record:
   - **Name**: `risus`
   - **Value**: your VM's public IPv4 address
   - **TTL**: `300`
4. Wait for propagation (usually under 5 minutes). Verify: `dig risus.example.com`

### 2. VM Firewall

Ports 80 and 443 must be reachable from the internet (Caddy needs port 80 for the Let's Encrypt ACME challenge). Port 8765 stays loopback-only — never expose it.

In Hetzner Cloud Console → Firewalls, allow inbound:

| Protocol | Port | Source |
|----------|------|--------|
| TCP | 80 | 0.0.0.0/0, ::/0 |
| TCP | 443 | 0.0.0.0/0, ::/0 |

### 3. Server Setup

On the VM, allow rootless Podman containers to bind the public HTTP/HTTPS
ports required by Caddy:

```bash
echo 'net.ipv4.ip_unprivileged_port_start=80' | sudo tee /etc/sysctl.d/99-rootless-low-ports.conf
sudo sysctl --system
```

This permits rootless containers to listen on ports 80 and 443 without running
the compose stack as root.

In the repo root with venv active:

```bash
export RISUS_TOKEN=your-secret-token-here   # min 16 chars — share with players
export DOMAIN=risus.example.com
```

Start the full stack including the Caddy TLS proxy:

**Podman:**

```bash
PATH=$PWD/.venv/bin:$PATH CONTAINER_ENGINE=podman podman-compose --profile production up -d --build
```

**Docker:**

```bash
docker compose --profile production up -d --build
```

Caddy automatically obtains a Let's Encrypt TLS certificate for
`risus.example.com` on first startup. This requires DNS propagation to be
complete, ports 80/443 open in the VM firewall, and low-port binding enabled
when using rootless Podman.

### 4. Verify

```bash
curl -fsS https://risus.example.com/healthz
# → {"ok":true}
```

### 5. Connect Clients

Bare hostname → client automatically uses `wss://` (TLS):

```bash
python risus.py risus.example.com YourName --token your-secret-token-here
```

Token travels encrypted inside the TLS connection. Caddy access logging is off by default — token never appears in logs.

### 6. Persist Token Across Reboots

Add to `/etc/environment` or a systemd drop-in so the token survives VM restarts:

```bash
# /etc/environment (persistent across reboots)
RISUS_TOKEN=your-secret-token-here
DOMAIN=risus.example.com
```

---

## Code Style & Commit Conventions

- Format: `black` (or equivalent)
- Commits: [Conventional Commits](https://www.conventionalcommits.org/) — `feat:`, `fix:`, `test:`, `docs:`, `chore:`
- See `AGENTS.md` for AI-agent rules and the hand-off checklist

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| "Connection rejected: invalid or missing token" | Token mismatch — ensure client `--token` matches `RISUS_TOKEN` on the server |
| Port 8765 already in use | `PATH=$PWD/.venv/bin:$PATH podman-compose down` or stop whatever owns 8765 |
| Podman SELinux mount error | Add `:Z` to the volume mount in `docker-compose.yml` for the SQL file |
| Schema init didn't run | Volume already exists; run `down -v` then `up -d` again |
| WS close code `4409 name in use` | Another client is already connected with that name; wait ~30 s or use a different name |
| E2E tests time out on CI | Ensure Docker daemon socket is accessible; for rootless Podman set `DOCKER_HOST` |
