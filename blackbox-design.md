# Black Box — Complete Design Document

## Overview

Black Box is a self-hosted diagnostic monitoring API that receives telemetry from applications, analyses it using a local LLM, raises GitHub issues when errors occur, and sends notifications via Telegram. It runs on a Raspberry Pi 5 (8GB) on a home network, exposed via Cloudflare Tunnel.

```
┌──────────────┐                ┌──────────────┐    local     ┌─────────┐
│  Your Apps   │──(registered)─▶│  Black Box   │────────────▶│ Ollama  │
│  (NuGet pkg) │   API key      │  ASP.NET API │              │ LLM     │
└──────────────┘                └──┬────┬────┬──┘             └─────────┘
       ▲                           │    │    │
       │ register + key            │    │    │
       └───────────────────────────┘    │    │
                                        │    │
                            GitHub API  │    │ Telegram API
                                 ▼           ▼
                            ┌────────┐  ┌──────────┐
                            │ GitHub │  │ Telegram  │
                            │ Issues │  │ 3x Bots   │
                            └────────┘  └──────────┘
```

---

## 1. Authentication

### Per-Install Registration

Apps register on first launch. The NuGet package handles this transparently. No secrets are baked into the package — the API key is generated at runtime and stored in the app's `appsettings.json`.

**Flow:**

```
FIRST LAUNCH:
  App calls POST /v1/register
    { appName, appVersion, machineId }
  Black Box generates a raw API key: "bb_a1b2c3d4..."
  Black Box stores SHA256(salt + key) in SQLite
  Black Box returns the raw key (only time it exists on the server)
  NuGet package saves the key to appsettings.json

SUBSEQUENT CALLS:
  Header: X-Api-Key: bb_a1b2c3d4...
  Black Box hashes the provided key and compares against stored hashes
```

**Key design:**

- `bb_` prefix for easy identification in logs
- Per-key salt prevents rainbow table attacks
- `CryptographicOperations.FixedTimeEquals` prevents timing attacks
- Keys are revocable via admin dashboard
- Optional approval flow: new registrations require admin approval before activation

**Machine ID** is derived from `SHA256(MachineName + OSVersion + UserName)` — stable, not personally identifiable.

**SQLite schema:**

```sql
CREATE TABLE api_keys (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    app_name        TEXT NOT NULL,
    machine_id      TEXT NOT NULL,
    key_hash        TEXT NOT NULL,
    salt            TEXT NOT NULL,
    created_at      DATETIME NOT NULL,
    last_used_at    DATETIME,
    is_revoked      INTEGER DEFAULT 0,
    revoked_at      DATETIME
);
```

---

## 2. LLM — Local on Pi

### Model

Llama 3.2 3B (or Qwen 2.5 3B) quantized to Q4, running via Ollama.

- ~3.5GB RAM for the model
- OpenAI-compatible API on `http://localhost:11434`
- JSON mode for structured output

**Memory budget:**

```
Raspberry Pi OS    ~0.5GB
Ollama + LLM       ~3.5GB
.NET runtime        ~0.1GB
Black Box API       ~0.1-0.2GB
Headroom            ~3.5GB
Total               ~8GB
```

### Structured Output

Ollama's JSON mode constrains the LLM to return valid JSON matching a schema:

```json
{
  "response": "string — conversational response to the app",
  "updatedSummary": "string — running session summary",
  "severity": "info | warning | error | critical",
  "action": {
    "type": "none | github_issue | github_comment",
    "title": "string",
    "body": "string"
  }
}
```

---

## 3. Session / Conversation Design

### Lifecycle

A session maps to one app run — from startup to shutdown (or timeout).

```
POST /v1/session/start     → returns sessionId + greeting
POST /v1/session/{id}/diagnostic  → repeated, returns analysis
POST /v1/session/{id}/end  → closes session, writes to memory
```

**Session timeout:** 30 minutes of inactivity auto-closes the session.

### Context Management — Sliding Window

The LLM can't hold all diagnostics in context. Solution: a running summary that the LLM updates on each call.

**What the LLM sees per call:**

```
┌─────────────────────────────────────────────────┐
│ SYSTEM PROMPT                                    │
│ • personality.md                                 │
│ • directives.md                                  │
│ • Relevant memories from memory.json             │
├─────────────────────────────────────────────────┤
│ SESSION CONTEXT                                  │
│ • App name, version, machine                     │
│ • Running summary (LLM-written, not raw logs)    │
│ • Last 2-3 raw diagnostics                       │
├─────────────────────────────────────────────────┤
│ CURRENT DIAGNOSTIC                               │
│ • The new error/log data + severity              │
├─────────────────────────────────────────────────┤
│ INSTRUCTIONS                                     │
│ • Analyse, update summary, decide action         │
└─────────────────────────────────────────────────┘
```

The running summary is how unlimited diagnostics fit in bounded context. The LLM maintains its own compressed memory of the session.

### Decision Flow

```
INFO     → Log, update summary. No external action.
WARNING  → Log, update summary, mention in response.
ERROR    → Create or update GitHub issue. Telegram notification.
CRITICAL → Create issue immediately. Urgent Telegram notification.

3+ related warnings in 5 minutes → escalate to ERROR.
```

### SQLite Schema

```sql
CREATE TABLE sessions (
    id              TEXT PRIMARY KEY,
    app_name        TEXT NOT NULL,
    app_version     TEXT,
    machine_id      TEXT,
    started_at      DATETIME NOT NULL,
    ended_at        DATETIME,
    status          TEXT DEFAULT 'active',
    running_summary TEXT
);

CREATE TABLE diagnostics (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id      TEXT REFERENCES sessions(id),
    received_at     DATETIME NOT NULL,
    severity        TEXT NOT NULL,
    category        TEXT,
    raw_data        TEXT NOT NULL,
    llm_analysis    TEXT,
    action_taken    TEXT
);
```

---

## 4. Three-File LLM Configuration

```
/blackbox/config/
├── personality.md      ← HOW it communicates
├── directives.md       ← WHAT it does, rules, thresholds
└── memory.json         ← WHAT it knows from experience
```

### personality.md

Pure character — tone, style, quirks. No rules or logic.

Defines: identity (name, creator, creation date), communication style, character notes. Edited whenever you want to adjust the voice.

### directives.md

The job description — severity classification, escalation rules, GitHub issue format, Telegram message format, session management rules, what NOT to do.

Edited when you want to tune behaviour — different thresholds, new rules, different formatting.

### memory.json

Two-layer system (see section 5).

---

## 5. Memory System

### Two Layers

```
LAYER 1: SQLite (total recall)
  Every diagnostic, every session, every response.
  Queryable. Complete record.

LAYER 2: memory.json (learned knowledge)
  Patterns, resolutions, identity, key facts.
  Small (<500 entries, <200KB). LLM-readable.
```

Everything that goes to the API is stored in SQLite. The memory file contains what the LLM has *learned* from processing that data — not the raw data itself.

### memory.json Structure

```json
{
  "identity": {
    "name": "Black Box",
    "created": "2026-05-19",
    "creator": "Toby Hunter",
    "organisation": "Nurtur Limited",
    "home": "Raspberry Pi 5, Toby's home network"
  },
  "entries": [
    {
      "id": "mem_001",
      "date": "2026-05-19",
      "category": "identity",
      "summary": "I was brought online for the first time.",
      "details": "...",
      "appRelated": null,
      "issueRef": null
    }
  ]
}
```

**Categories:** identity, issue_pattern, resolution, app_knowledge, instruction, stats_snapshot.

### Memory Creation Triggers

1. **Session end** — LLM reviews session, decides if anything new was learned
2. **Pattern recognition** — weekly job reviews data, spots trends
3. **Issue creation** — every GitHub issue gets a memory entry
4. **Your instructions** — things you tell it via Telegram
5. **Monthly summary** — aggregated stats snapshot

### Querying Memories

When you message Black Box via Telegram, it uses both layers:

- **SQLite** provides precise numbers (counts, dates, specific records)
- **memory.json** provides context and patterns
- **LLM** combines them into a useful conversational answer

Memory search is keyword-based (not vector/RAG) — adequate for a manageable memory file, no extra infrastructure needed.

### Size Management

**SQLite tiers:**

```
HOT (last 30 days)     → full raw diagnostics
WARM (30-180 days)     → errors kept, info/warning raw data cleared, metadata kept
COLD (180+ days)       → only session summaries + issues, individual diagnostics deleted
Estimated: ~50-100MB/year
```

**memory.json pruning (monthly):**

- LLM reviews entries older than 90 days
- Merges related entries (5 entries about same issue → 1 consolidated)
- Removes problem entries superseded by resolutions (resolution entry absorbs key facts first)
- Keeps: identity, active patterns, unresolved issues, stats snapshots, user instructions
- Target: <500 entries, <200KB

Resolution entries are never deleted. They absorb the facts from the problem entries they supersede, preserving the full story.

---

## 6. GitHub Issue Management

### Multi-Repo Support

Each app maps to its own repo. Configuration stored in SQLite, managed via admin dashboard.

```sql
CREATE TABLE repo_mappings (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    app_name    TEXT NOT NULL UNIQUE,
    owner       TEXT NOT NULL,
    repo        TEXT NOT NULL,
    labels      TEXT,
    created_at  DATETIME NOT NULL,
    updated_at  DATETIME NOT NULL
);
```

Fallback repo (`nurtur/blackbox-issues`) catches issues from unmapped apps. Telegram notification prompts you to add a mapping.

### Issue Creation

Uses Octokit (.NET GitHub client). GitHub PAT (fine-grained) scoped to specific repos with Issues read/write permission only.

**Issue format:**
- Title: concise, includes app name and error type
- Body: summary, stack trace, session context, LLM analysis, previous occurrences
- Labels: `black-box`, `app:{name}`, `severity:{level}`, optional area label

### Duplicate Prevention (4 layers)

1. **Local SQLite check** — open issue for this app + error category?
2. **LLM judgement** — same root cause as any open issue? (non-obvious duplicates)
3. **Rate limiting** — max 3 issues per app per hour
4. **Error signature** — SHA256 hash of exception type + top 3 stack frames. Same signature within 24 hours → append, don't create.

```sql
CREATE TABLE github_issues (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    github_number     INTEGER NOT NULL,
    owner             TEXT NOT NULL,
    repo              TEXT NOT NULL,
    app_name          TEXT NOT NULL,
    title             TEXT NOT NULL,
    status            TEXT DEFAULT 'open',
    error_signature   TEXT,
    root_cause        TEXT,
    created_at        DATETIME NOT NULL,
    closed_at         DATETIME,
    last_updated      DATETIME NOT NULL,
    occurrence_count  INTEGER DEFAULT 1
);
```

### Issue Sync

Every 15 minutes, Black Box checks open issues on GitHub. If closed:
- Update local status
- LLM creates a resolution memory entry (reads closing comment/PR for context)

---

## 7. Telegram — Three Bots

### Bot Separation

| Bot | Purpose | LLM-powered? | Two-way? |
|---|---|---|---|
| Black Box Alerts (`@blackbox_alerts_bot`) | App errors, GitHub issues, diagnostic summaries | Yes (personality.md) | Yes (chat) |
| Black Box Infrastructure (`@blackbox_infra_bot`) | Deploys, Pi health, disk space, Ollama status | No (template-based) | No (send-only) |
| Black Box Testing (`@blackbox_test_bot`) | E2E results, CI failures, test summaries | No (template-based) | No (send-only) |

### Two-Way Chat (App Bot Only)

Webhook registered only for the app alerts bot. You can:
- Ask questions: "How many issues for NurturCRM?"
- Give instructions: "Remember we're deploying v3 next Tuesday"
- Suppress alerts: "Ignore Portal warnings for the next hour"
- Get summaries: "What happened today?"

### Grafana Alerts → Infra Bot

Grafana notification channel points at the infra bot token/chat ID.

### CI Results → Test Bot

GitHub Actions workflows send results to the test bot token/chat ID.

---

## 8. NuGet Package — Private GitHub Packages

### Distribution

Source code in the public repo. Built package published to GitHub Packages (private). Apps add the private feed via `nuget.config` with a GitHub PAT (`read:packages` scope).

Published automatically via GitHub Action on release tags.

### Package Structure

```
BlackBox.Client/
├── BlackBoxClient.cs              ← Main entry point (fire-and-forget + async)
├── BlackBoxOptions.cs             ← Configuration model
├── Models/
│   ├── DiagnosticPayload.cs
│   ├── SessionResponse.cs
│   └── DiagnosticResponse.cs
├── Registration/
│   └── RegistrationHandler.cs     ← Auto-register on first run
├── Queue/
│   ├── BackgroundSender.cs        ← Drains queue, handles networking
│   ├── DiskQueue.cs               ← Persist unsent diagnostics to disk
│   └── RateLimiter.cs             ← Dedup burst errors
├── Extensions/
│   └── ServiceCollectionExtensions.cs
└── Middleware/
    └── BlackBoxExceptionMiddleware.cs
```

### Integration in Consuming Apps

```csharp
// Program.cs — two lines
builder.Services.AddBlackBox(builder.Configuration);
app.UseBlackBoxExceptionHandler();
```

```json
// appsettings.json
{
  "BlackBox": {
    "Endpoint": "https://blackbox.hunter-industries.co.uk",
    "AppName": "NurturCRM",
    "AppVersion": "2.1.0",
    "ApiKey": ""
  }
}
```

API key starts empty, populated automatically on first run.

### Two Send Methods

- `SendDiagnostic()` — fire and forget, returns void, instant. Use for most logging.
- `SendDiagnosticAsync()` — returns LLM analysis if wanted. 5s timeout, returns null on failure.

### Resilience

**Architecture:** Public API drops messages onto a `Channel<T>` in-memory queue and returns immediately. A background worker drains the queue and handles networking. The app is never blocked.

**Failure handling:**

| Scenario | Behaviour |
|---|---|
| Black Box is down | Queue to disk, health check every 30s, drain when back |
| Network timeout | 3s timeout per call, queued to disk |
| Registration fails | Retry with backoff (30s, 1m, 2m, 5m), queue to disk meanwhile |
| Session lost | Auto-start new session, retry the diagnostic |
| App closes normally | Flush queue, end session cleanly (3s grace period) |
| App crashes | Disk queue preserves unsent diagnostics, sent on next launch |
| Error burst | Rate limiter deduplicates within 5s window, bounded queue (1000) drops oldest if full |
| App offline | Disk queue up to 10MB, oldest dropped beyond that |

---

## 9. API Versioning

Major version only in URL prefix: `/v1/`

Full version returned in response header: `X-BlackBox-Version: 1.0.0`

NuGet package defaults to `ApiVersion = 1`, configurable if a v2 is ever released.

**All endpoints:**

```
POST   /v1/register
POST   /v1/session/start
POST   /v1/session/{id}/diagnostic
POST   /v1/session/{id}/end

GET    /v1/admin/status
GET    /v1/admin/status/ollama
GET    /v1/admin/status/github
GET    /v1/admin/status/telegram
GET    /v1/admin/keys
GET    /v1/admin/keys/{id}
POST   /v1/admin/keys/{id}/revoke
POST   /v1/admin/keys/{id}/approve
GET    /v1/admin/sessions
GET    /v1/admin/sessions/{id}
GET    /v1/admin/sessions/stats
GET    /v1/admin/issues
POST   /v1/admin/issues/sync
GET    /v1/admin/memory
POST   /v1/admin/memory/prune
DELETE /v1/admin/memory/{id}
GET    /v1/admin/logs
GET    /v1/admin/logs/stream

POST   /webhook/telegram  (no version — Telegram webhook)
GET    /metrics            (no version — Prometheus scrape)
```

---

## 10. Admin Dashboard

### Architecture

React frontend built with Vite. In production, built static files are served from the API's `wwwroot/` folder. Single process, single port, no CORS.

**Development:** React dev server (localhost:3000) proxies to API (localhost:5000).
**Production:** API serves everything on :5000. `/v1/*` → controllers. `/admin/*` → static files.

### Auth

Single admin user. Username/password login returns JWT. Credentials stored hashed in `appsettings.Local.json`.

### Pages

- **Dashboard** — active sessions, diagnostics today, issues this week, system status
- **Keys** — list all registered keys, usage stats, revoke, approve
- **Sessions** — active and recent sessions, drill into diagnostics
- **Issues** — local issue tracker mirror, force sync with GitHub
- **Memory** — view/delete entries, trigger pruning, export
- **Repos** — manage app-to-repo mappings, issue stats per repo
- **Logs** — live log tail via Server-Sent Events, filter by level/app
- **System** — health checks for API, Ollama, GitHub, Telegram

---

## 11. Error Handling & Resilience

### Principle

Black Box never loses diagnostic data and never blocks calling apps. If external services fail, data is stored and actions are queued for retry.

### Ollama Down

1. Store raw diagnostic in SQLite (always succeeds first)
2. Use rule-based fallback for severity (exception → ERROR, timeout/slow → WARNING, else → INFO)
3. If ERROR: create GitHub issue using template (no LLM summary)
4. Queue diagnostic for LLM reprocessing when Ollama recovers
5. Background job checks every 2 minutes, reprocesses pending items

### External Service Failures (GitHub, Telegram)

**Outbox pattern:** Failed actions saved to an outbox table, retried with exponential backoff (1m, 2m, 4m... max 1hr, max 10 retries).

```sql
CREATE TABLE outbox (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    action_type     TEXT NOT NULL,
    payload         TEXT NOT NULL,
    created_at      DATETIME NOT NULL,
    retry_count     INTEGER DEFAULT 0,
    next_retry_at   DATETIME,
    completed_at    DATETIME,
    error           TEXT
);
```

### Pi Reboot Recovery

On startup, Black Box marks all active sessions as "interrupted" and appends a note to their running summaries. NuGet package auto-starts a new session if it gets a 404 for its session ID.

### SQLite

WAL mode enabled. Busy timeout of 5 seconds. Daily automated backups via cron. Last 7 days of backups retained.

### Disk Monitoring

Background job checks hourly. Telegram infra alert if free space drops below 10%.

---

## 12. Logging & Observability

### Structured Logging (Serilog)

JSON-formatted logs. Rolling daily files, 14-day retention. Key events logged with structured fields (session ID, app name, duration, action taken).

### Prometheus Metrics

`prometheus-net` exposes `/metrics` endpoint. Grafana scrapes every 15 seconds.

**Metrics:**
- `blackbox_requests_total` (by endpoint, status)
- `blackbox_request_duration_seconds` (by endpoint)
- `blackbox_llm_duration_seconds`
- `blackbox_llm_failures_total`
- `blackbox_active_sessions`
- `blackbox_diagnostics_total` (by app, severity)
- `blackbox_issues_created_total` (by app)
- `blackbox_outbox_pending` (by action type)
- `blackbox_reprocess_queue_pending`
- `blackbox_database_size_bytes`
- `blackbox_memory_entries`

### Grafana Dashboard

Panels: diagnostics/hr, LLM latency, error rate, diagnostics by severity (stacked), active sessions, outbox/reprocess queue depth, system resources, service status.

**Grafana alerts** → infra Telegram bot:
- BlackBox API unreachable (2m)
- LLM failing consistently (5m)
- Outbox backlog growing (10m)
- Disk space below 10%

### Admin Log Viewer

Live log tail via Server-Sent Events. Filter by level and app. Last 100 entries loaded on page open.

---

## 13. Testing Strategy

### Three Layers

```
Unit tests          — no external dependencies, run every build (~80% of tests)
Integration tests   — real SQLite, mocked HTTP services, run every PR
E2E tests           — real services (test repo, test bot), run on release
```

### Test Infrastructure

- **Mock services** for LLM, GitHub, Telegram — record calls for assertions
- **In-memory SQLite** for integration tests — full isolation per test
- **WebApplicationFactory** for API integration tests
- **Dedicated test resources:** `nurtur/blackbox-test-issues` repo, test Telegram chat

### What to Test

- API key generation, validation, timing safety
- Error signature hashing and dedup logic
- Payload validation
- Memory search
- Full diagnostic flow (register → session → diagnostic → issue → notification)
- Failure modes (LLM down → fallback, GitHub down → outbox, malformed input → 400)
- Session lifecycle (start, timeout, recovery after reboot)
- Outbox retry and exponential backoff

### CI Pipeline

```yaml
push/PR:     Unit + integration tests
merge main:  Unit + integration + deploy API
release tag: Unit + integration + E2E + publish NuGet package
```

---

## 14. Deployment

### API Deployment (GitHub Action → Pi)

Triggered on push to main when API source files change.

1. Run tests
2. `dotnet publish -r linux-arm64`
3. Build React admin frontend, copy to `wwwroot/`
4. SCP to Pi staging folder
5. SSH: stop service → backup current → swap → restart
6. Health check (`/v1/admin/status`)
7. Pass → Telegram infra notification. Fail → rollback → Telegram alert.

**Pi directory structure:**

```
/home/pi/blackbox/
├── current/       ← live deployment
├── rollback/      ← previous version
├── config/        ← NEVER overwritten (personality.md, directives.md, memory.json)
├── data/          ← NEVER overwritten (blackbox.db)
├── logs/
└── backups/
```

Downtime per deployment: ~5 seconds.

### NuGet Package Publishing

Triggered on GitHub release creation. Builds, packs, pushes to GitHub Packages.

### SSH Access

Cloudflare Tunnel proxies SSH via a separate hostname (`ssh.hunter-industries.co.uk`). GitHub Action uses `cloudflared` to connect. No ports open on the router.

### Developing on Windows, Deploying to Linux

No special steps. Develop normally on Windows, publish with `-r linux-arm64`. Use `Path.Combine()` for file paths (not hardcoded slashes) and consistent file name casing.

`.gitattributes` normalises line endings to LF.

---

## 15. Hosting

### Raspberry Pi 5 (8GB, active cooling)

Three systemd services, all start on boot with auto-restart:

```
blackbox.service      ← API on port 5000
cloudflared.service   ← Cloudflare Tunnel
ollama.service        ← LLM on port 11434
```

### Cloudflare Tunnel

Pi connects outbound to Cloudflare. No open ports on the router. No port forwarding. Dynamic IP doesn't matter.

```yaml
# ~/.cloudflared/config.yml
ingress:
  - hostname: blackbox.hunter-industries.co.uk
    service: http://localhost:5000
  - hostname: ssh.hunter-industries.co.uk
    service: ssh://localhost:22
  - service: http_status:404
```

Cloudflare handles HTTPS termination. Pi serves plain HTTP. SSL mode: Flexible.

---

## 16. Repository Structure

Public GitHub repo. Private secrets gitignored.

```
blackbox/
├── src/
│   ├── BlackBox.Api/
│   │   ├── Controllers/
│   │   ├── Services/
│   │   ├── Middleware/
│   │   ├── Jobs/              ← background services (outbox, sync, reprocess, disk monitor)
│   │   ├── Data/              ← EF Core DbContext, migrations
│   │   ├── Program.cs
│   │   └── BlackBox.Api.csproj
│   │
│   ├── BlackBox.Client/
│   │   ├── BlackBoxClient.cs
│   │   ├── BlackBoxOptions.cs
│   │   ├── Models/
│   │   ├── Registration/
│   │   ├── Queue/
│   │   ├── Extensions/
│   │   ├── Middleware/
│   │   └── BlackBox.Client.csproj
│   │
│   └── BlackBox.Shared/
│       ├── DiagnosticPayload.cs
│       ├── SessionResponse.cs
│       ├── DiagnosticResponse.cs
│       └── BlackBox.Shared.csproj
│
├── tests/
│   ├── BlackBox.Unit.Tests/
│   ├── BlackBox.Integration.Tests/
│   └── BlackBox.E2E.Tests/
│
├── admin/                          ← React (Vite)
│   ├── src/
│   ├── package.json
│   └── vite.config.ts
│
├── config/
│   ├── personality.example.md
│   ├── directives.example.md
│   └── memory.example.json
│
├── .github/
│   └── workflows/
│       ├── test.yml
│       ├── deploy-api.yml
│       └── publish-client.yml
│
├── .gitignore
├── .gitattributes
├── BlackBox.sln
└── README.md
```

**Gitignored:**

```
appsettings.Local.json
config/personality.md
config/directives.md
config/memory.json
data/
```

---

## 17. GitHub Secrets

```
PI_HOST                      → blackbox.hunter-industries.co.uk
PI_USER                      → pi
PI_SSH_KEY                   → dedicated deploy SSH key

TELEGRAM_APP_BOT_TOKEN       → app alerts bot
TELEGRAM_APP_CHAT_ID         → app alerts chat
TELEGRAM_INFRA_BOT_TOKEN     → infrastructure bot
TELEGRAM_INFRA_CHAT_ID       → infrastructure chat
TELEGRAM_TEST_BOT_TOKEN      → testing bot
TELEGRAM_TEST_CHAT_ID        → testing chat

TEST_GITHUB_TOKEN            → PAT for test repo (E2E tests)
```

---

## 18. Technology Stack

| Component | Technology |
|---|---|
| API | ASP.NET Core 8, C# |
| Database | SQLite (EF Core) |
| LLM | Ollama + Llama 3.2 3B (Q4) |
| Admin frontend | React + Vite |
| NuGet hosting | GitHub Packages |
| GitHub integration | Octokit (.NET) |
| Notifications | Telegram Bot API (3 bots) |
| Metrics | prometheus-net |
| Logging | Serilog (JSON, rolling files) |
| Monitoring | Grafana + Prometheus |
| Hosting | Raspberry Pi 5 (8GB) |
| Tunnel | Cloudflare Tunnel (cloudflared) |
| CI/CD | GitHub Actions |
