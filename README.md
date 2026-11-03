<div align="center">
  <img src="assets/hero-banner.svg" alt="Zoth Relay (MCP Agent Bridge) — Sovereign Architecture Banner" width="100%" />
</div>

<br />

<div align="center">
  <a href="https://zoth.nullai.tech/tools/zoth-relay"><img src="https://img.shields.io/badge/Zoth_Studio_v2-Workspace-D4AF37?style=for-the-badge&logo=sparkles&logoColor=black" alt="Zoth Studio Workspace" /></a>
  <a href="https://zothos.nullai.tech"><img src="https://img.shields.io/badge/ZothOS_v3.3-Port_:8765-38BDF8?style=for-the-badge&logo=linux&logoColor=black" alt="Port :8765" /></a>
  <a href="https://nullai.tech"><img src="https://img.shields.io/badge/Security-100%25_Zero--Egress-10B981?style=for-the-badge&logo=shield&logoColor=black" alt="Zero Egress" /></a>
  <a href="https://buy.stripe.com/3cIfZg27PdKo2D64Ocfw40X"><img src="https://img.shields.io/badge/Support-☕_Buy_Us_a_Coffee-FFDD00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black" alt="Buy Us a Coffee" /></a>
</div>

# zoth-relay

A secure local bridge that lets **many AI agents** work on one ZOTHOS (Debian trixie / KDE) or Parrot OS machine, each with its own token, scope and audit trail. Supported clients include Meta Muse Code, Grok Bot, ChatGPT custom GPTs, Claude, Cursor and anything else that speaks MCP or HTTP.

```
 Muse Code ─┐  MCP (HTTP /mcp or stdio)
 Claude/Cursor ┤
 Grok Bot ──┤  REST /v1 (+ NDJSON stream)     cloudflared ──►  127.0.0.1:8765  zoth-relay (user zoth-relay, systemd-hardened)
 ChatGPT GPT ┤  GPT Actions (/openapi-actions.json)                    │
 dots, jev… ┘                                                     per-agent dirs, profile, rate limit, audit
```

zoth-relay evolved from `parrot-relay` and keeps everything it had: non-root service user, path sandbox, command blocklist, rate limits, JSON audit log, systemd hardening, cloudflared quick tunnel, and install/uninstall scripts.

## Install
```bash
sudo ./install.sh --with-cloudflared         # detects ZOTHOS / Parrot / Debian
sudo zoth-relay agent add muse --profile dev  # token printed ONCE
cloudflared tunnel --url http://localhost:8765
```
The first install creates a `local-admin` agent (read-only, `dashboard: true`) and prints its token. The **Zoth Relay** launcher in the KDE menu opens the read-only dashboard at `http://127.0.0.1:8765/dashboard`.

## Agents
```bash
sudo zoth-relay agent add NAME [--profile read-only|workspace|exec|dev] [--dir D]... [--allow-cmd PROG]... [--rate N] [--dashboard]
sudo zoth-relay agent list | revoke NAME | rotate NAME | remove NAME
```
| profile | status | list_dir / read_file | write_file | run_command |
|---|---|---|---|---|
| read-only | ✓ | ✓ | | |
| workspace | ✓ | ✓ | ✓ | |
| exec | ✓ | ✓ | ✓ | ✓ |
| dev | ✓ | ✓ | ✓ | ✓ (Landlock-sandboxed to one workspace, binds 127.0.0.1:4100-4199 only; see docs/muse-code.md) |
| mobile | ✓ | | | (only `POST /v1/ask` to configured AI targets; see MOBILE.md) |

`/etc/zoth-relay/agents.yaml` stores only the SHA-256 of each token. The service hot-reloads it, so `revoke` and `rotate` take effect on the next request. By default each agent gets `~/agent-workspace/<name>`, so agents can't see each other's files unless you give them a shared `--dir`.

## Interfaces
| | endpoint |
|---|---|
| REST | `/status /file /dir /exec /exec/run`, also under `/v1/...` |
| NDJSON stream | `POST /v1/exec` |
| MCP (streamable HTTP) | `POST /mcp`, tools `status list_dir read_file write_file run_command` |
| MCP (stdio) | `ZOTH_RELAY_TOKEN=… zoth-relay mcp-stdio`, a bridge to the running service |
| GPT Actions | `GET /openapi-actions.json` (OpenAPI 3.1, bearer, non-streaming) |
| Dashboard | `GET /dashboard`, loopback only + token with `dashboard: true` |
| Mobile ask | `GET /v1/agents`, `POST /v1/ask` (text + image, NDJSON stream), `POST /v1/live/token` (Gemini Live ephemeral token), `mobile` profile only. See [MOBILE.md](MOBILE.md) |
| Health | `GET /health` (alias of `/healthz`, public) |

Connection guides: [Grok Bot](docs/grok-bot.md) · [ChatGPT GPT Actions](docs/chatgpt-gpt-actions.md) · [Claude & Cursor](docs/claude-cursor-mcp.md) · [Muse Code](docs/muse-code.md) · [generic agents (dots, jev, …)](docs/generic-agent.md)

## Security model
Layers, from strongest to weakest:
1. **OS sandbox (the real protection).** The service runs as the `zoth-relay` system user: nologin, removed from sudo/adm/docker/disk, and its code is root-owned. systemd applies `NoNewPrivileges`, an empty capability set, `ProtectSystem=strict`, `ProtectHome=tmpfs` (other users' homes are invisible), `PrivateDevices`, `PrivateTmp`, kernel/cgroup protections, `RestrictNamespaces`, `MemoryMax=2G` and `TasksMax=256`.
2. **Per-agent tokens.** Each token is 288-bit random, and only its hash is stored. A token is never logged. Each agent has its own rate limit.
3. **Path scoping.** Every path is fully resolved, symlinks included, and must sit inside *that agent's* `allowed_dirs`. The same check applies to the exec working directory.
4. **Profiles and optional per-agent command allowlist.**
5. **Command blocklist.** Refuses `rm -rf /`, mkfs, dd to disks, sudo, shutdown, fork bombs and similar.
6. **Audit log.** `/var/log/zoth-relay/audit.log` records every request in JSON, tagged with `agent`, `via` (rest/mcp) and the IP (CF-Connecting-IP).

### Threat caveats (read these)
- **The blocklist and `--allow-cmd` can be bypassed.** They inspect a bash string, and bash is Turing-complete: `python3 -c`, `perl`, `eval "$(printf …)"`, scripts written to disk and then run, and so on. They exist to catch accidents and lazy prompt injection. They don't stop a determined attacker. **Any `exec` agent can do anything the `zoth-relay` user can do.** That includes reading every agent's directories, because they all live under one Unix user. Path scoping between agents is enforced only for the file APIs and the exec cwd, *not* inside a command. If agents must be truly isolated from each other, don't give the untrusted one `exec`; use the **`dev`** profile instead, which wraps every command in a Landlock kernel sandbox (own workspace read/write only, system dirs read-only, TCP bind limited to `dev_ports`), or run a second instance under a different user.
- Exec agents have network access (needed for git/pip). Anyone holding a token can use the box as a network pivot.
- A token is a password. Anyone who has it, including a prompt-injected agent, gets that agent's powers. Prefer `read-only` or `workspace`, and rotate tokens often.
- Quick tunnels publish an unauthenticated URL. The bearer token is the only gate, and `/openapi-actions.json` and `/healthz` are public (they contain no data). The dashboard refuses any request that carries Cloudflare headers.
- The MCP DNS-rebinding check is off unless you set `mcp_allowed_hosts`. The bearer token still applies.
- `mcp-stdio --direct` runs as *your* user and skips the systemd sandbox. It's for development only.
- Audit logs contain full command lines, so don't put secrets in commands.

## Development
```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements-dev.txt
.venv/bin/python -m pytest -q     # unit tests
./scripts/smoke.sh                # real server on 127.0.0.1:18765
gitleaks detect --source . -v     # secret scan
```

## ☕ Support 1nc0gn30 • NullAI (Zero Paywalls)

All tools in the Zoth Arsenal are **100% free, open, and unlocked by default**. If you enjoy this tool and want to support continued sovereign development:

- ☕ **Buy Us a Coffee (Stripe)**: [buy.stripe.com/3cIfZg27PdKo2D64Ocfw40X](https://buy.stripe.com/3cIfZg27PdKo2D64Ocfw40X)
- ◎ **Solana Tip Address**: `GG2eQbVh8ut1BK6j1A2tz7pSySdLpD38BgyEhGcxdXoK` (`solana:GG2eQbVh8ut1BK6j1A2tz7pSySdLpD38BgyEhGcxdXoK`)

---

<div align="center">
  <sub>Engineered by <b>1nc0gn30 • NullAI</b> (<a href="https://github.com/1nc0gn30">@1nc0gn30</a>) &bull; <a href="https://nullai.tech">NullAI Platform</a> &bull; <a href="https://zoth.nullai.tech">Zoth Studio v2</a></sub>
</div>

---

## 🏛️ ZothOS v3.4, Zoth Studio v2 & Obsidian Second Brain Ecosystem

```mermaid
flowchart LR
    subgraph ZothOS["🐧 ZothOS v3.4 CLI & Braille TUI"]
        CLI["zoth-relay --help / --json"]
        TUI["zoth-relay-tui<br/>Unicode U+2800 2x4 Braille"]
    end

    subgraph Workstation["⚡ Sovereign Workstation"]
        WEB["http://127.0.0.1:8765<br/>REST API & /health"]
    end

    subgraph Studio["🌌 Zoth Studio v2 & ZothOS-Lite"]
        GODMODE["http://127.0.0.1:3000/tools/zoth-relay<br/>God-Mode Workstation"]
        LITE["http://127.0.0.1:8145<br/>ZothOS-Lite Web Desktop"]
    end

    subgraph SecondBrain["💎 Obsidian Second Brain & STDP"]
        TRACKER["http://127.0.0.1:8151<br/>Zoth Obsidian Tracker"]
        NEURO["http://127.0.0.1:8094<br/>Neuro-Memory STDP Cortex"]
        NOTE["Studio/Tools/Studio zoth-relay.md"]
    end

    CLI --> WEB
    TUI --> WEB
    WEB <--> GODMODE
    WEB <--> LITE
    WEB --> TRACKER
    TRACKER <--> NOTE
    NOTE <--> NEURO
```

### Cross-Surface Integration Matrix

| Surface | Endpoint / Binary | Capabilities |
| :--- | :--- | :--- |
| **Standalone Workstation** | `http://127.0.0.1:8765` | 100% Free & Open-Access Web Deck, live `/health` & REST telemetry |
| **Zoth Studio v2 Route** | `http://127.0.0.1:3000/tools/zoth-relay` | Interactive God-Mode component inside Zoth Studio (`https://zoth.nullai.tech`) |
| **ZothOS v3.4 CLI & TUI** | `zoth-relay` / `zoth-relay-tui` | Global binary in `/usr/local/bin/` with Unicode `U+2800` Braille TUI & `--json` output |
| **ZothOS-Lite Web Desktop** | `http://127.0.0.1:8145` | Browser-native KDE-style window manager & terminal launcher |
| **Zoth Obsidian Tracker** | `http://127.0.0.1:8151` | Interactive Obsidian Second Brain vault explorer (`~/Documents/Obsidian Vault/Studio/Tools/Studio zoth-relay.md`) |
| **Neuro-Memory STDP Cortex** | `http://127.0.0.1:8094` | 3-Factor STDP synaptic memory daemon & 21-tool MCP server |

```bash
# Inspect CLI & Unicode U+2800 Braille TUI on ZothOS v3.4
zoth-relay --help
zoth-relay-tui --status
zoth-relay-tui --json

# Verify Second Brain Vault & 12-Step Verification Gallery on :8151
zoth-obsidian-tracker --verify
```

