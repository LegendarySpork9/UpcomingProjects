# Self-Hosted LLM — Full Implementation Plan

## Architecture Overview

```
┌─────────────┐       ┌──────────────────┐       ┌──────────────────────────────────────┐
│ Your device  │──────▶│ Cloudflare Edge   │──────▶│ GMKtec K8 Plus (Linux)               │
│ (anywhere)   │ HTTPS │                  │Tunnel │                                      │
│ + WARP       │       │ Zero Trust Access │       │ cloudflared                          │
│              │       │ ├─ Email OTP      │       │   └─▶ nginx                          │
│              │       │ ├─ WARP enrolled  │       │        ├─▶ React static files        │
│              │       │ └─ JWT injection  │       │        └─▶ .NET API (Kestrel :5000)  │
└─────────────┘       └──────────────────┘       │             ├─▶ Ollama (:11434)       │
                                                  │             ├─▶ SQLite                 │
                                                  │             ├─▶ JWT validation         │
                                                  │             └─▶ Rate limiting          │
                                                  │                                      │
                                                  │ RTX 3090 24GB (via Oculink)           │
                                                  │ Prometheus + node_exporter ──▶ Grafana │
                                                  └──────────────────────────────────────┘
```

---

## Phase 1: Hardware

### Purchase List

| # | Component | Specification | Est. Cost |
|---|-----------|--------------|-----------|
| 1 | GPU | NVIDIA RTX 3090 24GB (used) | £550-700 |
| 2 | Oculink cable | SFF-8612 to SFF-8612, 50cm | £10-15 |
| 3 | Oculink to PCIe adapter | ADT-Link or equivalent | £25-40 |
| 4 | PSU | 750W+ ATX PSU (e.g. Corsair RM750) | £60-80 |
| 5 | Open-air frame/bracket | Test bench frame or mount for GPU + adapter | £15-30 |

**Total: ~£660-865**

### Pre-Purchase Checks

- Verify K8 Plus Oculink port is externally accessible on the chassis
- Confirm PSU provides 2x 8-pin PCIe power connectors
- Plan physical location — 3090 is a large card (~30cm), open-air mount needs space and airflow

---

## Phase 2: Linux Setup

### 2.1 — OS Install on GMKtec

- Ubuntu Server 24.04 LTS (best NVIDIA driver support)
- Wipe Windows, clean install
- NVMe as boot/primary drive

### 2.2 — NVIDIA Drivers + CUDA

```bash
sudo apt install nvidia-driver-550 nvidia-cuda-toolkit
nvidia-smi  # verify GPU detected
```

### 2.3 — Verify eGPU via Oculink

- Connect hardware, boot, confirm `nvidia-smi` shows the 3090
- Check VRAM reported as 24GB

---

## Phase 3: Ollama

### 3.1 — Install

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 3.2 — Pull Models

```bash
ollama pull qwen2.5-coder:32b-instruct-q4_K_M   # primary coding model
ollama pull llama3.1:8b                           # fast general Q&A
ollama pull deepseek-coder-v2:16b                 # alternative coding model
```

### 3.3 — Verify

```bash
ollama run qwen2.5-coder:32b-instruct-q4_K_M "Write a hello world in C#"
```

### 3.4 — Configure

- Ollama listens on `localhost:11434` only (default — not exposed externally)
- Configure model unload timeout for idle VRAM management
- Runs as `ollama.service` via systemd (installed automatically)

---

## Phase 4: .NET API

### 4.1 — Project Setup

- ASP.NET Core Web API (.NET 8)
- EF Core with SQLite provider
- Hosted on the GMKtec, runs via Kestrel on `localhost:5000`

### 4.2 — Database Schema (SQLite)

```
Users
├── Id (GUID)
├── Email (unique — from Cloudflare JWT)
├── DisplayName
├── Role (admin/user)
├── DailyTokenLimit
├── RequestsPerMinuteLimit
├── CreatedAt
└── UpdatedAt

Conversations
├── Id (GUID)
├── UserId (FK → Users)
├── Title
├── Model
├── CreatedAt
└── UpdatedAt

Messages
├── Id (GUID)
├── ConversationId (FK → Conversations)
├── Role (user/assistant/system)
├── Content
├── TokensIn
├── TokensOut
├── ResponseTimeMs
└── CreatedAt

UserSettings
├── UserId (FK → Users)
├── Key
└── Value
```

### 4.3 — API Endpoints

| Method | Route | Purpose |
|--------|-------|---------|
| POST | `/api/chat` | Send message, stream response from Ollama |
| GET | `/api/conversations` | List user's conversations |
| GET | `/api/conversations/{id}` | Get conversation with messages |
| PUT | `/api/conversations/{id}` | Rename conversation |
| DELETE | `/api/conversations/{id}` | Delete conversation |
| GET | `/api/models` | List available Ollama models |
| GET | `/api/usage` | User's token usage stats |
| GET | `/api/metrics` | Prometheus-compatible metrics endpoint |
| GET | `/api/admin/users` | Admin: list all users and usage (admin role only) |
| PUT | `/api/admin/users/{id}/limits` | Admin: set rate limits per user (admin role only) |

### 4.4 — Auth (Cloudflare JWT Validation)

- Every request must include the `Cf-Access-Jwt-Assertion` header
- API validates the JWT against Cloudflare's public signing keys (`https://<team>.cloudflareaccess.com/cdn-cgi/access/certs`)
- Extract user email from the JWT claims
- Auto-create user record on first login
- Your email gets `admin` role by default, all others get `user` role

### 4.5 — Rate Limiting (Per User)

| Limit | Default (user) | Default (admin) |
|-------|---------------|-----------------|
| Requests per minute | 10 | Unlimited |
| Daily token budget | 50,000 | Unlimited |
| Max message length | 4,000 chars | Unlimited |

- Configurable per user via admin endpoints
- Returns `429 Too Many Requests` when exceeded
- Tracked in-memory (reset on restart is fine) for per-minute, in SQLite for daily

### 4.6 — Streaming

- The `/api/chat` endpoint streams the Ollama response using Server-Sent Events (SSE)
- Frontend receives tokens as they're generated, not waiting for full completion

### 4.7 — Systemd Service

```ini
[Unit]
Description=LLM Chat API
After=network.target ollama.service

[Service]
ExecStart=/usr/bin/dotnet /opt/llm-api/LlmApi.dll
WorkingDirectory=/opt/llm-api
Restart=always
RestartSec=5
User=llm-api

[Install]
WantedBy=multi-user.target
```

---

## Phase 5: React Frontend

### 5.1 — Tech Stack

| Layer | Choice |
|-------|--------|
| Framework | React + TypeScript |
| Build tool | Vite |
| Styling | Tailwind CSS |
| HTTP/streaming | `fetch` with `ReadableStream` (SSE) |
| Markdown rendering | `react-markdown` + `remark-gfm` |
| Code highlighting | `react-syntax-highlighter` |
| State management | Zustand |

### 5.2 — Features

| Feature | Detail |
|---------|--------|
| Chat interface | Message input with shift+enter for newlines, streaming response display |
| Markdown + code rendering | Full markdown support, syntax-highlighted code blocks with language detection |
| Code copy button | One-click copy on every code block |
| Conversation sidebar | List, select, rename, delete conversations |
| Model selector | Dropdown populated from `/api/models` |
| New conversation | Button to start fresh chat |
| Responsive layout | Works on desktop, tablet, and mobile |
| User display | Show logged-in user email from Cloudflare session |
| Usage indicator | Show token usage against daily budget (for non-admin users) |

### 5.3 — Layout

```
┌──────────────────────────────────────────────────┐
│  LLM Chat        Model: [Qwen 2.5 32B ▾]   user │
├────────────┬─────────────────────────────────────┤
│            │                                     │
│ + New Chat │  User: How do I parse JSON in C#?   │
│            │                                     │
│ Today      │  Assistant:                         │
│  Convo 1   │  You can use `System.Text.Json`:    │
│  Convo 2   │  ```csharp                          │
│            │  var obj = JsonSerializer            │
│ Yesterday  │    .Deserialize<MyType>(json);      │
│  Convo 3   │  ```                         [Copy] │
│  Convo 4   │                                     │
│            │                                     │
├────────────┴─────────────────────────────────────┤
│  [Type a message...                    ]  [Send] │
│  Tokens today: 12,340 / 50,000                   │
└──────────────────────────────────────────────────┘
```

### 5.4 — Build and Serve

- `npm run build` produces static files in `dist/`
- Deployed to `/var/www/llm-chat/` on the GMKtec
- Served by nginx as static files

---

## Phase 6: Networking

### 6.1 — nginx Config on GMKtec

```nginx
server {
    listen 80;
    server_name llm.yourdomain.com;

    location / {
        root /var/www/llm-chat;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_buffering off;           # important for SSE streaming
        proxy_cache off;
    }
}
```

### 6.2 — Cloudflare Tunnel

```bash
# Install
sudo apt install cloudflared

# Authenticate
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create llm-home

# Configure (~/.cloudflared/config.yml)
tunnel: <tunnel-id>
credentials-file: /root/.cloudflared/<tunnel-id>.json
ingress:
  - hostname: llm.yourdomain.com
    service: http://localhost:80
  # Add other services here (from Pi migration)
  - hostname: app.yourdomain.com
    service: http://192.168.1.x:80
  - service: http_status:404

# DNS
cloudflared tunnel route dns llm-home llm.yourdomain.com

# Install as service
sudo cloudflared service install
sudo systemctl enable cloudflared
```

---

## Phase 7: Auth & Device Lockdown

### 7.1 — Cloudflare Zero Trust (Dashboard Config)

| Setting | Value |
|---------|-------|
| Application name | LLM Chat |
| Application domain | `llm.yourdomain.com` |
| Session duration | 7 days |
| Identity provider | One-time PIN (email) |

### 7.2 — Access Policy

| Rule | Type | Value |
|------|------|-------|
| Allow | Email | `your-email@domain.com` |
| Allow | Email | (family members as added) |
| Require | Device posture | WARP enrolled in your Zero Trust org |
| Default | Block | Everything else |

### 7.3 — Device Enrolment

- Install Cloudflare WARP on each approved device (Windows, Mac, iOS, Android — all supported)
- Enrol against your Zero Trust org
- Unenrolled devices are blocked at the edge

### 7.4 — API-Level JWT Validation (Belt and Braces)

- .NET API validates `Cf-Access-Jwt-Assertion` on every request
- Rejects any request without a valid, correctly-signed JWT
- Even if someone somehow bypassed Cloudflare, the API won't serve them

---

## Phase 8: Monitoring & Operations

### 8.1 — Prometheus Stack

```bash
sudo apt install prometheus prometheus-node-exporter
# Add nvidia-gpu-exporter for GPU metrics
```

### 8.2 — Grafana Datasources

- Prometheus (system + GPU metrics)
- .NET API `/api/metrics` endpoint (app metrics)

### 8.3 — Grafana Dashboard

#### Electricity Cost Panels

| Panel | Calculation | Detail |
|-------|------------|--------|
| Live power draw | Real-time wattage from `nvidia-smi` | At-a-glance view of current GPU power consumption |
| Cost MTD (month to date) | Sum GPU power draw from 1st of month to now, convert to kWh, multiply by electricity rate | What you've spent this billing period |
| Projected cost (current month) | Cost MTD ÷ days elapsed × days in month | Extrapolates current usage to end of month |
| Cost over 12 months | Rolling 12-month sum of monthly costs | Long-term running cost visibility |

**PromQL examples:**

```promql
# Cost MTD — integrates power draw over time, converts to kWh, multiplies by rate
increase(nvidia_gpu_energy_consumed_joules[${__range}]) / 3600000 * ${electricity_rate}

# Average power approach (alternative)
avg_over_time(nvidia_gpu_power_draw_watts[${__range}]) * hours_elapsed / 1000 * ${electricity_rate}
```

**Grafana variables:**

| Variable | Value | Notes |
|----------|-------|-------|
| `electricity_rate` | `0.245` | £/kWh — update when your tariff changes |
| System idle draw | ~50W (GMKtec + peripherals) | Optional: include whole-system cost, not just GPU |

Optional: add a smart plug with power monitoring (e.g. Tapo P110 ~£12) for true whole-system power measurement, feeding into Prometheus via Home Assistant or similar.

#### Full Dashboard Layout

| Section | Panels |
|---------|--------|
| **Cost** | Live power draw · Cost MTD · Projected this month · 12-month rolling cost |
| **GPU** | Temperature · VRAM usage · Power draw over time |
| **Usage** | Tokens per day (per user) · Response time (avg, p95) · Active model |
| **System** | CPU · RAM · Disk · Tunnel status |
| **Alerts** | GPU overheating · Service down · Tunnel disconnected · Disk space |

### 8.4 — Grafana Alerts

| Alert | Condition |
|-------|-----------|
| GPU overheating | Temperature > 85°C for 5 min |
| Service down | Any systemd service in failed state |
| Tunnel disconnected | Tunnel down > 5 min |
| Disk space | Usage > 80% |

### 8.5 — Backup (Cron)

```bash
# /etc/cron.d/llm-backup — daily at 3am
0 3 * * * llm-api sqlite3 /var/lib/llm-chat/chat.db ".backup /backups/llm-chat/chat-$(date +\%F).db"
```

- Retain 30 days of backups, rotate older ones
- Optional: sync to Cloudflare R2 for off-site backup

### 8.6 — Maintenance Schedule

| Task | Frequency |
|------|-----------|
| Check for Ollama model updates | Monthly |
| Ubuntu security updates | Automatic via `unattended-upgrades` |
| NVIDIA driver updates | Only when needed, test after |
| Review Grafana dashboards / alerts | Monthly |
| Review Cloudflare Access logs | As needed |
| SQLite backup verification | Monthly (restore a backup, check integrity) |

---

## Implementation Order

| Step | Task | Dependencies |
|------|------|-------------|
| 1 | Purchase hardware (GPU, Oculink adapter, PSU, frame) | None |
| 2 | Install Ubuntu Server on GMKtec | None |
| 3 | Install NVIDIA drivers, connect eGPU, verify | Step 1, 2 |
| 4 | Install and test Ollama with models | Step 3 |
| 5 | Build .NET API (schema, endpoints, JWT validation, rate limiting) | Step 4 for testing |
| 6 | Build React frontend | Step 5 for API integration |
| 7 | Configure nginx | Step 5, 6 |
| 8 | Set up Cloudflare Tunnel + Zero Trust Access | Step 7 |
| 9 | Enrol devices with WARP | Step 8 |
| 10 | Set up Prometheus + Grafana monitoring | Step 4+ |
| 11 | Set up backups | Step 5 |
| 12 | Migrate existing services from Pi tunnel | Step 8 |

---

## Cost Summary

| Category | Cost | Frequency |
|----------|------|-----------|
| Hardware (GPU, Oculink, PSU, frame) | £660-865 | One-time |
| Electricity (est. 4hrs/day usage) | £7-10 | Monthly |
| Software (Ollama, Cloudflare, Prometheus, Grafana) | £0 | Free/open-source |
| Cloudflare Zero Trust | £0 | Free tier (up to 50 users) |
