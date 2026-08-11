# Server Backup Tool — API Project Plan

## Overview

An ASP.NET Core Web API that provides remote monitoring and control of game servers managed by the Server Backup Tool (SBT). The API and SBT communicate through a shared SQLite database with two core tables: **Logs** (SBT writes, API reads) and **Commands** (API writes, SBT reads). Both sides poll the database every second to detect changes. The API sends webhooks to a configurable listener whenever new logs appear, and the SBT executes and removes commands as they arrive.

Each SBT instance is uniquely identified by its server name (from App.config). Only one server with a given name can run at any one time, so the server name is the natural key for filtering logs and commands across multiple installs.

```
                         ┌───────────────────┐
                         │  Listening Service │
                         │   (webhook sink)   │
                         └────────▲───────────┘
                                  │ POST (new logs)
                                  │
┌──────────────┐          ┌───────┴────────┐          ┌──────────────┐
│   SBT #1     │          │    SBT API     │          │   SBT #2     │
│  (Console)   │          │  (ASP.NET Core)│          │  (Console)   │
│              │          │                │          │              │
│ Writes Logs  │          │ Reads Logs     │          │ Writes Logs  │
│ Reads Cmds   │◄────────►│ Writes Cmds    │◄────────►│ Reads Cmds   │
│ Deletes Cmds │          │ Reads Archives │          │ Deletes Cmds │
│              │          │                │          │              │
└──────┬───────┘          └───────┬────────┘          └──────┬───────┘
       │                          │                          │
       └──────────────────┬───────┴──────────────────────────┘
                          ▼
                   ┌─────────────┐
                   │  SQLite DB  │
                   │   sbt.db    │
                   │  (WAL mode) │
                   └─────────────┘
```

---

## 1. Solution Structure

Add a new project to the existing SBT solution:

```
Server-Backup-Tool/
├── Server Backup Tool/                    # Existing console app (modified)
├── Server Backup Tool.Tests/              # Existing test project (extended)
├── Server Backup Tool.Api/                # NEW — ASP.NET Core Web API
│   ├── Program.cs                         # Entry point, service registration
│   ├── appsettings.json                   # Database path, webhook URL, polling
│   ├── Controllers/
│   │   ├── LogsController.cs             # GET logs (live + archived)
│   │   └── CommandsController.cs         # POST commands, GET command status
│   ├── Authentication/
│   │   ├── ClientAuthHandler.cs         # AuthenticationHandler implementation
│   │   └── ClientAuthOptions.cs         # Options for client ID/secret auth
│   ├── Services/
│   │   ├── LogPollingService.cs          # Background polling for new logs
│   │   ├── WebhookService.cs            # Sends POST to listening service
│   │   ├── ArchiveReaderService.cs      # Extracts and reads archived logs
│   │   └── DatabaseService.cs           # SQLite connection + query helpers
│   ├── Models/
│   │   ├── LogEntry.cs                  # Log table row model
│   │   ├── CommandEntry.cs              # Command table row model
│   │   ├── WebhookPayload.cs            # Webhook POST body
│   │   └── Requests/
│   │       ├── CreateCommandRequest.cs  # POST command body
│   │       └── GetLogsRequest.cs        # Query parameters for log filtering
│   └── Server Backup Tool.Api.csproj
├── Server Backup Tool.Api.Tests/          # NEW — API unit tests
│   ├── Controllers/
│   │   ├── LogsControllerTests.cs
│   │   └── CommandsControllerTests.cs
│   ├── Services/
│   │   ├── LogPollingServiceTests.cs
│   │   ├── WebhookServiceTests.cs
│   │   └── ArchiveReaderServiceTests.cs
│   └── Server Backup Tool.Api.Tests.csproj
```

---

## 2. Dependencies

### API Project

| Package | Purpose |
|---|---|
| `Microsoft.Data.Sqlite` | SQLite database access |
| `Microsoft.AspNetCore.OpenApi` | Swagger/OpenAPI documentation |
| `Swashbuckle.AspNetCore` | Swagger UI for development |

### SBT Changes (additions to existing project)

| Package | Purpose |
|---|---|
| `Microsoft.Data.Sqlite` | SQLite database access for writing logs and reading commands |

No ORM is used. Raw SQL keeps the footprint small and the queries explicit.

---

## 3. SQLite Database Schema

Two tables, both keyed by `ServerName`. The SBT's server name from its App.config `serverDetails` element is used as the filter — only one server with a given name runs at any time, so the name is a natural unique identifier.

```sql
PRAGMA journal_mode=WAL;

CREATE TABLE IF NOT EXISTS Logs (
    Id          INTEGER PRIMARY KEY AUTOINCREMENT,
    ServerName  TEXT    NOT NULL,
    Timestamp   TEXT    NOT NULL,
    Level       TEXT    NOT NULL,
    Logger      TEXT    NOT NULL,
    Message     TEXT    NOT NULL
);

CREATE TABLE IF NOT EXISTS Commands (
    Id          INTEGER PRIMARY KEY AUTOINCREMENT,
    ServerName  TEXT    NOT NULL,
    Target      TEXT    NOT NULL,
    Command     TEXT    NOT NULL,
    CreatedAt   TEXT    NOT NULL
);

CREATE INDEX IF NOT EXISTS IX_Logs_Server ON Logs (ServerName, Id);

CREATE INDEX IF NOT EXISTS IX_Commands_Server ON Commands (ServerName);
```

### Access Patterns

| Actor | Table | Operations |
|---|---|---|
| SBT | Logs | INSERT (writes every log message) |
| SBT | Commands | SELECT by ServerName, DELETE after execution |
| API | Logs | SELECT (filtered by ServerName, with pagination) |
| API | Commands | INSERT (creates new commands) |

### Concurrency

WAL mode allows multiple readers and a single writer concurrently. Since the SBT and API perform short, fast writes, contention will be minimal. Both sides should use a busy timeout of 5 seconds (`PRAGMA busy_timeout=5000`) to handle any brief lock waits.

---

## 4. SBT Modifications

### 4.1 New Abstraction — `IDatabaseService`

```
Server Backup Tool/
├── Abstractions/
│   └── IDatabaseService.cs              # NEW
├── Implementations/
│   └── DatabaseService.cs               # NEW — SQLite implementation
├── Services/
│   └── Command Polling Service.cs       # NEW — polls and executes commands
```

**`IDatabaseService`** exposes:

| Method | Purpose |
|---|---|
| `WriteLog(string level, string logger, string message)` | Insert a row into the Logs table |
| `GetPendingCommands()` | Select all commands for this server |
| `DeleteCommand(int commandId)` | Delete a command after execution |
| `ClearLogs()` | Delete all Logs rows for this server |

The SBT's server name is read from the existing `serverDetails` `name` attribute in App.config at startup. Every database call scopes to this name.

### 4.2 Log Writing Integration

The existing `LoggerService` writes to log4net. To also write to SQLite:

1. **`LoggerServiceWrapper`** receives an `IDatabaseService` dependency via constructor injection
2. Each log method (`LogMessage`, `LogServerMessage`) calls `IDatabaseService.WriteLog()` after writing to log4net
3. The database write is fire-and-forget on a background thread so it never blocks logging
4. If the database write fails, the failure is logged to log4net only (no recursion)

### 4.3 Command Polling Service

A new `CommandPollingService` runs on a 1-second `System.Timers.Timer` (consistent with the existing timer pattern in `TimerService`):

1. Every second, call `IDatabaseService.GetPendingCommands()`
2. For each command, check the `Target` field to determine where to route it:
   - **Server** — pass the `Command` string directly to the game server's stdin via `ServerService`
   - **Tool** — execute the `Command` as an internal SBT operation
3. Call `IDatabaseService.DeleteCommand(commandId)` to remove it from the table
4. If execution fails, log the error but still delete the command to avoid retry loops

**Command structure:**

Each command has two parts:

| Field | Description | Examples |
|---|---|---|
| `Target` | Where the command is routed | `Server`, `Tool` |
| `Command` | The exact string to execute on the target | `/say hello`, `stop`, `backup`, `archive` |

**Target: Server**

The `Command` value is written directly to the game server's stdin. This allows any server command to be sent remotely without the SBT needing to know the full command set for each game. Examples:

| Command | Effect |
|---|---|
| `stop` | Stops the Minecraft server |
| `/say Server restarting in 5 minutes` | Broadcasts a message to players |
| `/whitelist add PlayerName` | Adds a player to the whitelist |
| `/op PlayerName` | Grants operator status |

**Target: Tool**

The `Command` value triggers an internal SBT operation. The supported tool commands are:

| Command | Action |
|---|---|
| `stop` | Sends the game-specific stop command to the server via `ServerConverter.StopCommand()` then closes the SBT |
| `restart` | Stops the server then re-launches via `ServerService` |
| `backup` | Triggers an immediate world backup via `JobService.RunJobs("backup")` |
| `archive` | Triggers log archival via `JobService.RunJobs("archive")` |
| `clean` | Triggers cleanup of old files via `JobService.RunJobs("clean")` |

### 4.4 Log Table Clearing on Archive

When `JobService.RunJobs("archive")` runs, after archiving the log files, it calls `IDatabaseService.ClearLogs()` to delete all Logs rows for this server. This ensures the Logs table always reflects the current (non-archived) log state.

### 4.5 App.config Addition

A new configuration element for the database:

```xml
<serverBackup>
    <serverDetails name="..." game="..." location="..." startFile="..." ipAddress="..." />
    <databaseDetails path="C:\ProgramData\Hunter Industries\Server Backup Tool\Data\sbt.db" pollingInterval="1000" />
    <!-- existing timerDetails, notifications -->
</serverBackup>
```

| Attribute | Type | Default | Description |
|---|---|---|---|
| `path` | string | `%PROGRAMDATA%\Hunter Industries\Server Backup Tool\Data\sbt.db` | Absolute path to the shared SQLite database |
| `pollingInterval` | int | 1000 | Milliseconds between command polls |

The existing `name` attribute on `serverDetails` is used as the server's identity in all database operations. No additional ID is needed.

---

## 5. API Design

### 5.1 Configuration — `appsettings.json`

```json
{
  "Authentication": {
    "ClientId": "<SHA-512 hash of the client ID>",
    "ClientSecret": "<SHA-512 hash of the client secret>"
  },
  "Database": {
    "Path": "C:\\ProgramData\\Hunter Industries\\Server Backup Tool\\Data\\sbt.db",
    "PollingIntervalMs": 1000
  },
  "Webhook": {
    "Url": "https://example.com/webhook/sbt-logs",
    "TimeoutSeconds": 10,
    "RetryCount": 3
  },
  "ArchiveSettings": {
    "ArchiveDirectory": ".\\Archived Logs"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

### 5.2 Authentication

All API endpoints require authentication via a client ID and client secret. The credentials are stored in `appsettings.json` as SHA-512 hashes — the plaintext values are never stored and cannot be recovered from the config.

**Request format:**

The caller provides credentials in the `Authorization` header using the `Basic` scheme. The header value is a Base64-encoded string of `{clientId}:{clientSecret}`:

```
Authorization: Basic <Base64(clientId:clientSecret)>
```

**Verification flow:**

1. Extract the `Authorization` header from the incoming request
2. Decode the Base64 value and split on `:` to get the plaintext client ID and client secret
3. Hash both values independently with SHA-512
4. Compare the resulting hashes against the `Authentication:ClientId` and `Authentication:ClientSecret` values stored in `appsettings.json`
5. If both match, the request is authenticated. If either fails, return 401 Unauthorized

**Implementation:**

- `ClientAuthHandler` extends `AuthenticationHandler<ClientAuthOptions>` and performs the hash-and-compare logic
- Registered in `Program.cs` via `builder.Services.AddAuthentication().AddScheme<ClientAuthOptions, ClientAuthHandler>()`
- All controllers are decorated with `[Authorize]`

Because the credentials are stored as one-way hashes, there is no way to retrieve the original client ID or secret from the configuration file. The installer warns the user to copy these values during setup (see installer plan).

### 5.3 Endpoints

#### Logs

| Method | Route | Description |
|---|---|---|
| `GET` | `/api/servers/{serverName}/logs` | Get live logs for a server |
| `GET` | `/api/servers/{serverName}/logs/archived` | List available archived log files |
| `GET` | `/api/servers/{serverName}/logs/archived/{fileName}` | Extract and return logs from a specific archive |

**`GET /api/servers/{serverName}/logs`** Query Parameters:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `level` | string | (all) | Filter by log level (Info, Warn, Error, Debug) |
| `logger` | string | (all) | Filter by logger name (ToolLogs, ServerLogs) |
| `limit` | int | 100 | Maximum number of rows to return |
| `afterId` | int | (none) | Cursor-based pagination — return logs with Id > afterId |

Response:

```json
{
  "serverName": "Minecraft-1",
  "logs": [
    {
      "id": 1042,
      "timestamp": "2026-08-11T14:30:00Z",
      "level": "Info",
      "logger": "ToolLogs",
      "message": "Server started successfully."
    }
  ],
  "nextAfterId": 1042
}
```

**`GET /api/servers/{serverName}/logs/archived`** Response:

```json
{
  "serverName": "Minecraft-1",
  "archives": [
    {
      "fileName": "Server 01-08-2026.zip",
      "createdAt": "2026-08-01T03:00:00Z",
      "sizeBytes": 245760
    }
  ]
}
```

**`GET /api/servers/{serverName}/logs/archived/{fileName}`** Response:

Returns the log file contents extracted from the archive. The API extracts the archive to a temporary directory, reads all log files, combines the content, deletes the extracted files, and returns the combined content in a single response.

```json
{
  "serverName": "Minecraft-1",
  "archiveName": "Server 01-08-2026.zip",
  "logs": [
    {
      "fileName": "Server.log",
      "content": "2026-08-01 00:00:01 INFO - Server started...\n..."
    }
  ]
}
```

#### Commands

| Method | Route | Description |
|---|---|---|
| `POST` | `/api/servers/{serverName}/commands` | Send a command to a server |
| `GET` | `/api/servers/{serverName}/commands` | List pending commands for a server |

**`POST /api/servers/{serverName}/commands`** Request:

```json
{
  "target": "Server",
  "command": "/say hello"
}
```

Response:

```json
{
  "id": 7,
  "serverName": "Minecraft-1",
  "target": "Server",
  "command": "/say hello",
  "createdAt": "2026-08-11T14:35:00Z"
}
```

**`GET /api/servers/{serverName}/commands`** Query Parameters:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `limit` | int | 50 | Maximum number of rows to return |

**Validation:**

- `target` must be `Server` or `Tool` (case-insensitive). Any other value is rejected with 400 Bad Request.
- `command` must be a non-empty string.
- When `target` is `Tool`, the `command` must be one of the recognised tool commands (`stop`, `restart`, `backup`, `archive`, `clean`). Unknown tool commands are rejected with 400 Bad Request.
- When `target` is `Server`, any non-empty `command` is accepted — the SBT passes it through to the game server's stdin as-is.

---

## 6. Background Services

### 6.1 Log Polling Service

A hosted `BackgroundService` that runs for the lifetime of the API:

1. Every second (configurable via `PollingIntervalMs`), query the Logs table for each known server name
2. Track the last seen `Id` per server name in memory
3. If new rows exist (Id > last seen), collect them into a webhook payload
4. Send the payload to the configured webhook URL via `WebhookService`
5. Update the last seen Id

On startup, the service reads the current max Id per server name so it doesn't replay historical logs. The list of known server names is derived from the distinct `ServerName` values in the Logs table.

```
Poll cycle:
  ┌─────────────────────────────────────────────┐
  │ SELECT Id, ServerName, Timestamp, Level,    │
  │        Logger, Message                      │
  │ FROM Logs                                   │
  │ WHERE Id > @lastSeenId                      │
  │   AND ServerName = @serverName              │
  │ ORDER BY Id ASC                             │
  └──────────────────────┬──────────────────────┘
                         │
                    New rows?
                    ┌────┴────┐
                   Yes       No
                    │         │
                    ▼         ▼
             Send webhook   Sleep 1s
             Update cursor
                    │
                    ▼
               Sleep 1s
```

### 6.2 Webhook Service

Sends HTTP POST requests to the configured listener URL.

**Payload format:**

```json
{
  "serverName": "Minecraft-1",
  "timestamp": "2026-08-11T14:30:05Z",
  "logs": [
    {
      "id": 1043,
      "timestamp": "2026-08-11T14:30:00Z",
      "level": "Info",
      "logger": "ToolLogs",
      "message": "Backup completed successfully."
    },
    {
      "id": 1044,
      "timestamp": "2026-08-11T14:30:01Z",
      "level": "Info",
      "logger": "ServerLogs",
      "message": "[14:30:01 INFO]: Saving chunks..."
    }
  ]
}
```

**Retry policy:**

| Attempt | Delay |
|---|---|
| 1st retry | 2 seconds |
| 2nd retry | 4 seconds |
| 3rd retry | 8 seconds |

After 3 failed retries, log the failure and move on. The logs remain in the database, so the webhook URL can query them later via the API if needed. Failed webhook deliveries do not block the polling loop.

### 6.3 Archive Reader Service

Handles the extraction and reading of archived log files:

1. Receive a request for a specific archive file (e.g., `Server 01-08-2026.zip`)
2. Verify the file exists in the archive directory
3. Extract to a temporary directory under `%TEMP%`
4. Read all extracted log files into memory
5. Delete the temporary directory and all extracted files
6. Return the combined log content

The extraction is done in a `try/finally` block to guarantee cleanup even if reading fails. The archive directory path comes from a configured default in `appsettings.json`.

---

## 7. SBT Command Execution Flow

When the SBT polls and finds a command:

```
┌──────────────────────────────────────────────────────────┐
│                  Command Polling Cycle                    │
│                                                          │
│  1. SELECT * FROM Commands                               │
│     WHERE ServerName = @serverName                       │
│                                                          │
│  2. For each row, check Target:                          │
│     ┌──────────────────────────────────────────────────┐ │
│     │                                                  │ │
│     │  Target = "Server"                               │ │
│     │    → Write Command to game server stdin           │ │
│     │      e.g. "/say hello" → stdin                   │ │
│     │                                                  │ │
│     │  Target = "Tool"                                 │ │
│     │    ├── stop     → StopServer() + close SBT       │ │
│     │    ├── restart  → Stop + Restart                 │ │
│     │    ├── backup   → JobService.RunJobs("backup")   │ │
│     │    ├── archive  → JobService.RunJobs("archive")  │ │
│     │    └── clean    → JobService.RunJobs("clean")    │ │
│     │                                                  │ │
│     │  DELETE FROM Commands WHERE Id = @commandId      │ │
│     └──────────────────────────────────────────────────┘ │
│                                                          │
│  3. Sleep 1 second                                       │
│                                                          │
│  4. Repeat                                               │
└──────────────────────────────────────────────────────────┘
```

Commands are deleted regardless of execution outcome. If a command fails (e.g., the server is already stopped when a `stop` tool command arrives), the error is logged to both log4net and the Logs table. The API consumer can observe the outcome via the logs.

---

## 8. Log Lifecycle

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│ Server runs │     │ Logs table   │     │ API reads logs  │
│ SBT writes  │────►│ accumulates  │────►│ sends webhooks  │
│ logs to DB  │     │ for server   │     │ to listener     │
└─────────────┘     └──────┬───────┘     └─────────────────┘
                           │
                    Archive trigger
                           │
                           ▼
                ┌──────────────────────┐
                │ 1. Archive log files │
                │ 2. Clear Logs table  │
                │    for this server   │
                │ 3. Fresh logs begin  │
                └──────────────────────┘
                           │
                    Later, via API:
                           │
                           ▼
                ┌──────────────────────┐
                │ GET archived/{file}  │
                │ 1. Extract ZIP       │
                │ 2. Read contents     │
                │ 3. Delete extracted  │
                │ 4. Return content    │
                └──────────────────────┘
```

When the archive job runs:
1. Log files are compressed into a ZIP in `.\Archived Logs\`
2. Original log files are deleted (existing behaviour)
3. **NEW:** `IDatabaseService.ClearLogs()` deletes all Logs rows for this server name
4. The Logs table starts fresh — the API's `LogPollingService` resets its cursor for this server
5. The webhook listener receives a notification that logs were archived (via the archive job's own log messages)

---

## 9. Error Handling

| Scenario | Behaviour |
|---|---|
| Missing or invalid Authorization header | API returns 401 Unauthorized |
| Client ID or secret hash mismatch | API returns 401 Unauthorized |
| Database file missing | API returns 503 Service Unavailable. SBT logs error and continues without DB features |
| Database locked (busy) | Both sides use `busy_timeout=5000`. If still locked after 5s, log error and retry on next poll |
| Unknown command string | API rejects at POST time with 400 Bad Request |
| Command execution failure | SBT logs the error, deletes the command |
| Webhook delivery failure | Log error, retry up to 3 times with exponential backoff, then skip |
| Archive file not found | API returns 404 Not Found |
| Archive extraction failure | API returns 500, `finally` block cleans up any partial extraction |
| Server name has no logs | API returns 200 with an empty logs array |

---

## 10. Build Order

| Phase | Work | Depends On |
|---|---|---|
| 1 | Add `Microsoft.Data.Sqlite` to the SBT project. Create `IDatabaseService` and `DatabaseService`. Wire up connection with busy timeout and WAL mode | Nothing |
| 2 | Add `databaseDetails` element to the App.config configuration model | Nothing |
| 3 | Modify `LoggerServiceWrapper` to write logs to SQLite via `IDatabaseService` alongside log4net | Phases 1, 2 |
| 4 | Create `CommandPollingService` with 1-second timer, command parsing, and execution routing through existing services | Phase 1 |
| 5 | Modify `JobService.RunJobs("archive")` to call `IDatabaseService.ClearLogs()` after archiving | Phase 1 |
| 6 | Scaffold the API project — `Program.cs`, `appsettings.json`, `DatabaseService`, Swagger setup | Nothing |
| 7 | Implement `ClientAuthHandler` and `ClientAuthOptions` — SHA-512 hash-and-compare authentication, wire into the pipeline | Phase 6 |
| 8 | Implement `LogsController` — GET live logs with filtering and pagination | Phase 7 |
| 9 | Implement `CommandsController` — POST commands with validation, GET pending commands | Phase 7 |
| 10 | Implement `ArchiveReaderService` and archived log endpoints on `LogsController` | Phase 7 |
| 11 | Implement `LogPollingService` background service with per-server cursor tracking | Phase 6 |
| 12 | Implement `WebhookService` with retry policy, integrate with `LogPollingService` | Phase 11 |
| 13 | Write unit tests for SBT changes — `DatabaseService`, `CommandPollingService`, log clearing | Phases 3, 4, 5 |
| 14 | Write unit tests for API — controllers, auth handler, polling service, webhook service, archive reader | Phases 7-12 |
| 15 | Integration testing — end-to-end flow with real SQLite database, verify polling, command execution, webhook delivery, authentication | Phases 13, 14 |
