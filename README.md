# zoth-relay

Sandboxed multi-agent HTTP, NDJSON streaming, and Model Context Protocol (MCP) relay daemon for ZothOS (Debian Trixie / KDE Plasma) and Parrot OS workstations on `127.0.0.1:8765`.

`zoth-relay` allows multiple autonomous AI clients (Claude Code, Cursor, Hermes, custom GPT Actions, mobile wearables, and local scripts) to operate on one Linux host with isolated workspaces, SHA-256 hashed bearer tokens, Landlock kernel sandboxing, per-agent rate limits, and append-only JSON audit logging.

---

## Architecture

```
Claude / Cursor / Hermes  ─┐  MCP (Streamable HTTP /mcp or stdio)
Custom GPT Actions        ─┤  OpenAPI 3.1 (/openapi-actions.json)
CLI & Swarm Workers       ─┤  REST /v1 + NDJSON stream (/v1/exec)
Wearable / Mobile Clients ─┘  Tailscale mobile profile (:8766 /v1/ask)
                                      │
                                      ▼
                    127.0.0.1:8765  zoth-relay (FastAPI + ASGI MCPGuard)
                    ├── SHA-256 token registry (/etc/zoth-relay/agents.yaml, hot-reload)
                    ├── Realpath symlink-aware directory confinement
                    ├── Landlock kernel filesystem + port sandbox (dev profile)
                    ├── Systemd service confinement (NoNewPrivileges, ProtectSystem=strict)
                    └── Structured audit trail (/var/log/zoth-relay/audit.log)
```

---

## Quick Start & Installation

### System Service Installation (ZothOS / Debian / Parrot)

```bash
git clone https://github.com/1nc0gn30/zoth-relay.git
cd zoth-relay
sudo ./install.sh --with-cloudflared
sudo zoth-relay agent add hermes --profile dev
```

On initial installation, `install.sh` provisions an unprivileged `zoth-relay` system user, generates `/etc/zoth-relay/config.yaml`, creates a read-only `local-admin` token (`dashboard: true`), and starts `zoth-relay.service` bound to `127.0.0.1:8765`.

### Local Development & Test Suite

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements-dev.txt
.venv/bin/python -m pytest -q
./scripts/smoke.sh
```

---

## Agent Profiles & Token Lifecycle

```bash
sudo zoth-relay agent add NAME [--profile read-only|workspace|exec|dev|mobile] \
  [--dir PATH]... [--allow-cmd BIN]... [--rate REQ_PER_MIN] [--dashboard]

sudo zoth-relay agent list
sudo zoth-relay agent revoke NAME
sudo zoth-relay agent rotate NAME
sudo zoth-relay agent remove NAME
```

| Profile | `status` | `list_dir` / `read_file` | `write_file` | `run_command` | Isolation Notes |
| :--- | :---: | :---: | :---: | :---: | :--- |
| `read-only` | Yes | Yes | No | No | Read-only inspection within `allowed_dirs` |
| `workspace` | Yes | Yes | Yes | No | File read/write within `allowed_dirs`, no shell execution |
| `exec` | Yes | Yes | Yes | Yes | Non-interactive bash with timeout and command blocklist |
| `dev` | Yes | Yes | Yes | Yes | Landlock-confined to agent workspace; TCP bind restricted to `127.0.0.1:4100-4199` |
| `mobile` | Yes | No | No | No | `POST /v1/ask` and `/v1/live/token` only; zero host file or shell access (see [MOBILE.md](MOBILE.md)) |

`/etc/zoth-relay/agents.yaml` stores only the SHA-256 digest of each 288-bit bearer token. Token revocations and rotations hot-reload on the next incoming request without restarting the daemon.

---

## Protocol Surfaces

| Surface | Endpoint / Command | Description |
| :--- | :--- | :--- |
| **REST v1** | `GET /v1/status`, `GET /v1/dir`, `GET /v1/file`, `POST /v1/file`, `POST /v1/exec/run` | Synchronous JSON file & command operations |
| **NDJSON Exec Stream** | `POST /v1/exec` | Real-time `stdout` / `stderr` / `exit_code` line streaming |
| **MCP Streamable HTTP** | `POST /mcp` | Exposes `status`, `list_dir`, `read_file`, `write_file`, `run_command` (MCP SDK 1.x & 2.x compatible) |
| **MCP Stdio Bridge** | `ZOTH_RELAY_TOKEN=zr_... zoth-relay mcp-stdio` | Bridges local stdio MCP hosts to the hardened `:8765` service |
| **OpenAPI 3.1 Actions** | `GET /openapi-actions.json` | Validated OpenAPI 3.1 schema for HTTP tool-calling clients |
| **Loopback Dashboard** | `GET /dashboard` | Loopback-only (`127.0.0.1` / `::1`) status view requiring `dashboard: true` token; rejects Cloudflare headers |
| **Health Probe** | `GET /health`, `GET /healthz` | Unauthenticated liveness check (`{"ok": true}`) |

Client configuration guides:
- [Claude & Cursor MCP](docs/claude-cursor-mcp.md)
- [ChatGPT GPT Actions](docs/chatgpt-gpt-actions.md)
- [Muse Code (`dev` Landlock profile)](docs/muse-code.md)
- [Grok Bot](docs/grok-bot.md)
- [Generic HTTP Agents](docs/generic-agent.md)
- [Mobile & Wearable API](MOBILE.md)

---

## Security Model & Operational Caveats

1. **Systemd & Unix User Confinement**: Runs as `zoth-relay` (`nologin`, stripped of `sudo`/`adm`/`docker`/`disk` groups) under `NoNewPrivileges=yes`, `ProtectSystem=strict`, `ProtectHome=tmpfs`, `PrivateDevices=yes`, `PrivateTmp=yes`, `RestrictNamespaces=yes`, `MemoryMax=2G`, and `TasksMax=256`.
2. **Realpath Path Confinement**: Every requested file path and `working_directory` is canonicalized (`Path.resolve()`) and checked against that specific agent's `allowed_dirs`, blocking `../` traversal and symlink escapes.
3. **Exec vs. Landlock `dev` Profile**: Command blocklists and `--allow-cmd` inspect shell strings and catch accidental destructive invocations, not adversarial shell escapes. Untrusted code-writing agents should always be assigned `--profile dev` (which enforces kernel-level Landlock filesystem and port restrictions) rather than `exec`.
4. **Audit Hygiene**: `/var/log/zoth-relay/audit.log` records timestamps, agent identifiers, protocol (`rest` or `mcp`), client IPs, and command arguments, while never logging bearer tokens.

---

## License & Support

- **License**: MIT
- **Open Access**: 100% free and open-source with zero paywalls or feature gates.
- **Support**: [Buy Us a Coffee via Stripe](https://buy.stripe.com/3cIfZg27PdKo2D64Ocfw40X) · **Solana Tip Address**: `GG2eQbVh8ut1BK6j1A2tz7pSySdLpD38BgyEhGcxdXoK`

Engineered by **Neal Frazier Tech (@1nc0gn30)** ([GitHub (zoth-relay)](https://github.com/1nc0gn30/zoth-relay)) · [NullAI Platform](https://nullai.tech) · [Zoth Studio](https://zoth.nullai.tech)

## Sovereign Surfaces

_Machine-generated surface inventory (docs/tools/layerD_probe.py). Do not hand-edit; re-run the probe instead.

**What it does.** Zoth Relay (MCP Agent Bridge) — Zoth Relay (MCP Agent Bridge) (v0.5.0) — Hardened local MCP/HTTP bridge so many AI agents share one ZothOS machine with per-agent tokens, path sandbox, command blocklist, rate limits, and audit trail on 127.0.0.1:8765.

**Category.** Swarm & Core  ·  **Version.** 0.5.0  ·  **Execution type.** `local_cli`

**Architecture & port.** Canonical source: `~/NullAITech/zoth-relay`. Assigned `localPort` **8765** (`http://127.0.0.1:8765`).

**ZothOS CLI usage.**

```bash
zoth-relay --help     # exit 0
zoth-relay --json     # exit 0
```

**Zoth Studio route.** https://zoth.nullai.tech/tools/zoth-relay

_Verified 2026-11-15 across the three install surfaces (`/usr/local/bin`, `~/.local/bin`, `zoth-os/config/includes.chroot/usr/local/bin`)._
