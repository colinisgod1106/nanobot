# Deployment

Use this page after `nanobot agent -m "Hello!"` works locally. Deployment keeps long-running surfaces online: WebUI, chat apps, heartbeat, Dream, cron jobs, and channel connections.

## Before You Deploy

Check these once before Docker, systemd, or LaunchAgent:

| Check | Why it matters |
|---|---|
| `nanobot status` shows the expected config and workspace | Confirms the process will read the instance you meant to run |
| `nanobot agent -m "Hello!"` works | Proves install, config, provider, model, and workspace writes before adding a service layer |
| Secrets are in environment variables or protected config files | API keys, bot tokens, OAuth state, and chat credentials should not be world-readable |
| `~/.nanobot/` or your custom config/workspace path is persistent | Sessions, memory, channel login state, generated artifacts, and cron jobs live there |
| Channel access control is intentional | Use `allowFrom`, pairing, WebSocket `token`/`tokenIssueSecret`, or private test channels before exposing the bot |
| Ports are planned | Gateway health defaults to `18790`; WebUI/WebSocket defaults to `8765`; `nanobot serve` defaults to `8900` |
| Logs are easy to reach | Use `docker compose logs`, `journalctl`, LaunchAgent log files, or `nanobot gateway --verbose` while diagnosing startup |

Restart the deployed process after editing `config.json`. Long-running processes read config at startup.

## Choose a Runtime

| Runtime | Use it for | State location | Useful first command |
|---|---|---|---|
| Docker Compose | Repeatable container runs on Linux servers or workstations | Bind-mount `~/.nanobot` to `/home/nanobot/.nanobot` | `docker compose run --rm nanobot-cli agent -m "Hello!"` |
| Docker CLI | Manual container testing or small one-off hosts | Bind-mount `~/.nanobot` to `/home/nanobot/.nanobot` | `docker run -v ~/.nanobot:/home/nanobot/.nanobot --rm nanobot status` |
| systemd user service | Linux user-level gateway that restarts automatically | Host user's `~/.nanobot` unless you pass explicit paths | `systemctl --user status nanobot-gateway` |
| macOS LaunchAgent | macOS gateway that starts after login | Host user's `~/.nanobot` unless the plist passes explicit paths | `launchctl list | grep ai.nanobot.gateway` |

## Docker

> [!TIP]
> The `-v ~/.nanobot:/home/nanobot/.nanobot` flag mounts your local config directory into the container, so your config and workspace persist across container restarts.
> The container runs as the non-root user `nanobot` (UID 1000) and reads config from `/home/nanobot/.nanobot`. Always mount your host config directory to `/home/nanobot/.nanobot`, not `/root/.nanobot`.
> If you get **Permission denied**, fix ownership on the host first: `sudo chown -R 1000:1000 ~/.nanobot`, or pass `--user $(id -u):$(id -g)` to match your host UID. Podman users can use `--userns=keep-id` instead.
>
> [!IMPORTANT]
> Official Docker usage currently means building from this repository with the included `Dockerfile`. Docker Hub images under third-party namespaces are not maintained or verified by HKUDS/nanobot; do not mount API keys or bot tokens into them unless you trust the publisher.

> [!IMPORTANT]
> The gateway and WebSocket channel default to `host: "127.0.0.1"` in `config.json` (set in `nanobot/config/schema.py`). Docker `-p` port forwarding cannot reach a container's loopback interface, so for the host or LAN to reach the exposed ports you must set both binds to `0.0.0.0` in `~/.nanobot/config.json` before starting the container. To serve the bundled WebUI from Docker, enable the WebSocket channel and protect bootstrap with a secret:
>
> ```json
> {
>   "gateway": { "host": "0.0.0.0" },
>   "channels": {
>     "websocket": {
>       "enabled": true,
>       "host": "0.0.0.0",
>       "port": 8765,
>       "tokenIssueSecret": "your-secret-here"
>     }
>   }
> }
> ```
>
> When the WebSocket `host` is `0.0.0.0`, the channel refuses to start unless `token` or `tokenIssueSecret` is also configured. See [`webui.md#lan-access`](./webui.md#lan-access) for details.

### Docker Compose

```bash
docker compose run --rm nanobot-cli onboard   # first-time setup
vim ~/.nanobot/config.json                     # add API keys
docker compose up -d nanobot-gateway           # start gateway
```

```bash
docker compose run --rm nanobot-cli agent -m "Hello!"   # run CLI
docker compose logs -f nanobot-gateway                   # view logs
docker compose down                                      # stop
```

### Docker

```bash
# Build the image
docker build -t nanobot .

# Initialize config (first time only)
docker run -v ~/.nanobot:/home/nanobot/.nanobot --rm nanobot onboard

# Edit config on host to add API keys
vim ~/.nanobot/config.json

# Run gateway (connects to enabled channels, e.g. Telegram/Discord/Mochat).
# Mirrors the security caps and port mappings declared in docker-compose.yml:
#   - `--cap-drop ALL --cap-add SYS_ADMIN` + unconfined apparmor/seccomp are required
#     when `tools.exec.sandbox: "bwrap"` is enabled (bwrap needs CAP_SYS_ADMIN for
#     user namespaces). Without them, `bwrap` exits with `clone3: Operation not permitted`.
#   - `-p 8765:8765` exposes the WebSocket channel / WebUI alongside the gateway health
#     endpoint on 18790.
docker run \
  --cap-drop ALL --cap-add SYS_ADMIN \
  --security-opt apparmor=unconfined \
  --security-opt seccomp=unconfined \
  -v ~/.nanobot:/home/nanobot/.nanobot \
  -p 18790:18790 -p 8765:8765 \
  nanobot gateway

# Or run a single command
docker run -v ~/.nanobot:/home/nanobot/.nanobot --rm nanobot agent -m "Hello!"
docker run -v ~/.nanobot:/home/nanobot/.nanobot --rm nanobot status
```

## Linux Service

Run the gateway as a systemd user service so it starts automatically and restarts on failure.

Preview the generated unit first:

```bash
nanobot gateway install-service --manager systemd --dry-run
```

Install, enable, and start it:

```bash
nanobot gateway install-service --manager systemd
```

For a custom instance, pass the same config/workspace selector you use to run the gateway:

```bash
nanobot gateway install-service \
  --manager systemd \
  --name nanobot-telegram \
  --config ~/.nanobot-telegram/config.json \
  --workspace ~/.nanobot-telegram/workspace
```

Common operations:

```bash
systemctl --user status nanobot-gateway        # check status
systemctl --user restart nanobot-gateway       # restart after config changes
journalctl --user -u nanobot-gateway -f        # follow logs
nanobot gateway uninstall-service --manager systemd
```

The installer writes `~/.config/systemd/user/nanobot-gateway.service`, runs
`systemctl --user daemon-reload`, enables the unit, and restarts it. It uses the
current Python executable with `python -m nanobot gateway --foreground`, so the
service runs in the same environment you used to install nanobot.

> **Note:** User services only run while you are logged in. To keep the gateway running after logout, enable lingering:
>
> ```bash
> loginctl enable-linger $USER
> ```

## macOS LaunchAgent

Use a LaunchAgent when you want `nanobot gateway` to stay online after you log in, without keeping a terminal open.

Preview the generated plist first:

```bash
nanobot gateway install-service --manager launchd --dry-run
```

Install, load, enable, and start it:

```bash
nanobot gateway install-service --manager launchd
```

For a custom instance:

```bash
nanobot gateway install-service \
  --manager launchd \
  --name nanobot-telegram \
  --config ~/.nanobot-telegram/config.json \
  --workspace ~/.nanobot-telegram/workspace
```

Common operations:

```bash
launchctl list | grep ai.nanobot.gateway
launchctl kickstart -k gui/$(id -u)/ai.nanobot.gateway
nanobot gateway uninstall-service --manager launchd
```

The installer writes `~/Library/LaunchAgents/ai.nanobot.gateway.plist`, uses the
current Python executable with `python -m nanobot gateway --foreground`, and
writes LaunchAgent logs under `~/.nanobot/logs/`.

> **Note:** if startup fails with "address already in use", stop the manually started `nanobot gateway` process first.

## Railway

[Railway](https://railway.com) is a cloud deployment platform that builds from your GitHub repo and runs containers with minimal configuration.

### Prerequisites

- A GitHub repository with nanobot pushed to it
- A Railway account (free trial: $5 one-time credit, 30 days)
- At least one LLM provider API key (OpenAI, Anthropic, etc.)

### Step 1: Connect GitHub

1. Go to [railway.com](https://railway.com) → **New Project** → **Deploy from GitHub repo**
2. Select your nanobot repository
3. Railway auto-detects the `Dockerfile` and starts building

### Step 2: Set Environment Variables

Railway automatically injects the `Dockerfile`'s `EXPOSE` ports. Add these **Service Variables** in the Railway dashboard:

| Variable | Value | Required |
|---|---|---|
| `NANOBOT_GATEWAY__HOST` | `0.0.0.0` | Yes — binds health endpoint to all interfaces |
| `NANOBOT_PROVIDERS__OPENAI__API_KEY` | `sk-...` | At least one provider key |
| `NANOBOT_AGENTS__DEFAULTS__MODEL` | `openai/gpt-4o` | No — defaults to `anthropic/claude-opus-4-5` |

Nanobot reads all config from `NANOBOT_*` environment variables via `pydantic-settings`. The model string follows the `provider/model` convention.

**Example — OpenAI:**

| Variable | Value |
|---|---|
| `NANOBOT_PROVIDERS__OPENAI__API_KEY` | `sk-...` |
| `NANOBOT_AGENTS__DEFAULTS__MODEL` | `openai/gpt-4o` |

**Example — Anthropic:**

| Variable | Value |
|---|---|
| `NANOBOT_PROVIDERS__ANTHROPIC__API_KEY` | `sk-ant-...` |
| `NANOBOT_AGENTS__DEFAULTS__MODEL` | `anthropic/claude-sonnet-4-20250514` |

**Example — DeepSeek (custom base URL):**

| Variable | Value |
|---|---|
| `NANOBOT_PROVIDERS__DEEPSEEK__API_KEY` | `sk-...` |
| `NANOBOT_PROVIDERS__DEEPSEEK__BASE_URL` | `https://api.deepseek.com` |
| `NANOBOT_AGENTS__DEFAULTS__MODEL` | `deepseek/deepseek-chat` |

**Additional optional variables:**

| Variable | Default | Description |
|---|---|---|
| `NANOBOT_AGENTS__DEFAULTS__MAX_TOKENS` | `8192` | Max output tokens |
| `NANOBOT_AGENTS__DEFAULTS__TEMPERATURE` | `0.1` | Generation temperature |
| `NANOBOT_AGENTS__DEFAULTS__REASONING_EFFORT` | (none) | `low`, `medium`, or `high` |
| `NANOBOT_WORKSPACE_SANDBOX_ENFORCED` | `true` | Set to `false` on free tier (no `CAP_SYS_ADMIN`) |
| `NANOBOT_CHANNELS__WEBSOCKET__HOST` | `127.0.0.1` | Set to `0.0.0.0` to expose WebUI |
| `NANOBOT_CHANNELS__WEBSOCKET__PORT` | `8765` | WebSocket/WebUI port |
| `NANOBOT_CHANNELS__WEBSOCKET__TOKEN_ISSUE_SECRET` | (none) | Required when WebSocket host is `0.0.0.0` |

### Step 3: Add a Volume (Recommended)

A Volume persists sessions, memory, and channel login state across deploys.

1. In your Railway project, go to **Volumes** → **Add Volume**
2. Set mount path to `/home/nanobot/.nanobot`
3. Attach it to your nanobot service

Without a Volume, all data is lost on redeploy (ephemeral storage).

### Step 4: Configure Networking and Health Check

| Setting | Value |
|---|---|
| **Public port** | `8765` — WebSocket channel / WebUI |
| **Health check port** | `18790` |
| **Health check path** | `/health` |

1. Go to your service → **Settings** → **Networking**
2. Add a **Public Port** for `8765` to expose the WebSocket/WebUI
3. Set the **Health Check** to port `18790`, path `/health`
4. Railway marks the deployment `Active` only after `/health` returns `{"status": "ok"}`

### First Deploy

After setting variables, Railway builds and starts the container. Check the **Deploy Logs** for:

```
✓ Health endpoint: http://0.0.0.0:18790/health
```

If the container restarts repeatedly, verify that at least one provider API key is set and the sandbox is disabled (free tier lacks `CAP_SYS_ADMIN`).

### Free Tier Limitations

| Limitation | Impact |
|---|---|
| **$5 trial credit** | Lasts ~1-2 weeks for a lightweight agent. After depletion, services stop. Upgrade to Hobby ($5/mo) to continue. |
| **1 GB RAM, 1 vCPU** | Enough for nanobot but may throttle under heavy tool calls. |
| **Sleep on idle** | WebSocket connections drop and cron jobs stop during sleep. Cold start on next request. |
| **No `CAP_SYS_ADMIN`** | `bwrap` sandbox cannot create user namespaces. Set `NANOBOT_WORKSPACE_SANDBOX_ENFORCED=false` or disable exec sandbox in config. |
| **1 GB ephemeral disk** | Sessions and memory fill the root disk unless a Volume is mounted. |
| **Peak hour deploy blocks** | Free tier cannot deploy during 8 AM – 8 PM local peak hours. |

### Upgrade to Hobby Plan

To remove trial credit expiry, sleep, and peak-hour restrictions:

1. Go to Railway dashboard → **Upgrade** → **Hobby** ($5/month)
2. No code changes needed — the same container keeps running
