# Server Backup Tool — Installer Project Plan

## Overview

A standalone .NET TUI console application using **Spectre.Console** that installs, configures, and updates the Server Backup Tool (SBT) and optionally the SBT API. The installer handles file deployment, App.config generation, SQLite database creation, and scheduled task registration.

```
┌─────────────────────────────────────┐
│         SBT Installer (TUI)         │
│         Spectre.Console App         │
├─────────────────────────────────────┤
│                                     │
│  1. Select Components               │
│     [x] Server Backup Tool          │
│     [ ] SBT API                     │
│                                     │
│  2. Configure Install Location      │
│  3. Configure SBT Settings          │
│  4. Install Files                   │
│  5. Post-Install Setup              │
│                                     │
└─────────┬──────────────┬────────────┘
          │              │
          ▼              ▼
   ┌─────────────┐ ┌──────────┐
   │ SBT Console │ │ SBT API  │
   │    App      │ │ (IIS)    │
   │             │ │          │
   └──────┬──────┘ └────┬─────┘
          │              │
          └──────┬───────┘
                 ▼
          ┌─────────────┐
          │  SQLite DB   │
          │   sbt.db     │
          └─────────────┘
```

---

## 1. Solution Structure

Add a new project to the existing SBT solution:

```
Server-Backup-Tool/
├── Server Backup Tool/                  # Existing console app
├── Server Backup Tool.Tests/            # Existing test project
├── Server Backup Tool.Installer/        # NEW — Spectre.Console TUI
│   ├── Program.cs                       # Entry point, mode detection
│   ├── Modes/
│   │   ├── InstallMode.cs              # Fresh install flow
│   │   ├── UpdateMode.cs              # Update existing installation
│   │   ├── ConfigureMode.cs           # Edit existing configuration
│   │   └── UninstallMode.cs           # Remove installation
│   ├── Steps/
│   │   ├── ComponentSelectionStep.cs  # SBT + optional API
│   │   ├── LocationStep.cs            # Install directory picker
│   │   ├── ServerConfigStep.cs        # Server details (name, game, path, IP)
│   │   ├── TimerConfigStep.cs         # Backup time + custom timers
│   │   ├── EmailConfigStep.cs         # SMTP + recipients
│   │   ├── ApiConfigStep.cs           # API port, binding, database path
│   │   ├── FileDeployStep.cs          # Copy published files
│   │   ├── ConfigGenerationStep.cs    # Generate App.config from inputs
│   │   ├── DatabaseSetupStep.cs       # Create SQLite DB, WAL, tables
│   │   ├── ScheduledTaskStep.cs       # Register SBT with Task Scheduler
│   │   └── ValidationStep.cs          # Post-install checks
│   ├── Services/
│   │   ├── ConfigWriter.cs            # Builds App.config XML
│   │   ├── DatabaseInitialiser.cs     # SQLite setup
│   │   ├── TaskSchedulerService.cs    # Windows Task Scheduler integration
│   │   ├── VersionService.cs          # Version comparison for updates
│   │   └── FileService.cs             # File copy, permissions, cleanup
│   ├── Models/
│   │   ├── InstallOptions.cs          # Collected user choices
│   │   ├── ServerConfig.cs            # Server details model
│   │   ├── TimerConfig.cs             # Timer details model
│   │   └── EmailConfig.cs             # Email/SMTP model
│   └── Server Backup Tool.Installer.csproj
```

---

## 2. Dependencies

| Package | Purpose |
|---|---|
| `Spectre.Console` | TUI framework — prompts, tables, progress bars, panels |
| `Microsoft.Data.Sqlite` | SQLite database creation and table setup |
| `System.Xml.Linq` | Programmatic App.config XML generation |
| `Microsoft.Win32.TaskScheduler` (optional) | Task Scheduler API wrapper |

No additional runtime prerequisites beyond .NET 6.0.

---

## 3. Mode Detection

The installer runs in one of three modes, determined by command-line arguments:

```
SBTInstaller.exe                  → Interactive mode selector
SBTInstaller.exe --install        → Jump straight to install
SBTInstaller.exe --update         → Jump straight to update
SBTInstaller.exe --configure      → Jump straight to configure
SBTInstaller.exe --uninstall      → Jump straight to uninstall
```

If no argument is provided, the TUI presents a selection prompt:

```
┌─────────────────────────────────────┐
│   Server Backup Tool — Setup        │
│   Version 2.0.2                     │
└─────────────────────────────────────┘

What would you like to do?

> Install Server Backup Tool
  Update an existing installation
  Configure an existing installation
  Uninstall Server Backup Tool
```

---

## 4. Install Flow

### Step 1 — Component Selection

```
Select components to install:

[x] Server Backup Tool (required)
[ ] SBT API
```

SBT is always selected and cannot be deselected. The API is optional.

### Step 2 — Install Location

```
Install location: C:\Program Files\Hunter Industries\Server Backup Tool
> Use default
  Choose custom location
```

The installer validates write permissions to the selected directory. If the directory does not exist, the installer creates it.

### Step 3 — Server Configuration

Collect the values needed for the `serverDetails` section of App.config:

| Prompt | Input Type | Validation |
|---|---|---|
| Server name | Text | Required, non-empty |
| Game type | Selection list | Minecraft (extendable later) |
| Server directory | Text (path) | Directory must exist, must contain the start file |
| Start file | Text | File must exist relative to server directory |
| Server IP address | Text | Valid IPv4 address format |

### Step 4 — Backup and Timer Configuration

Collect the values needed for the `timerDetails` section:

| Prompt | Input Type | Validation |
|---|---|---|
| Daily backup time | Text | Valid HH:mm:ss format |
| Time zone | Text | Optional, valid timezone ID |
| Add custom timers? | Yes/No | — |
| For each timer: name, time, message | Text | Time in HH:mm:ss, message non-empty |

### Step 5 — Email Configuration

```
Enable email notifications? [y/n]
```

If yes:

| Prompt | Input Type | Validation |
|---|---|---|
| SMTP server hostname | Text | Required |
| SMTP port | Number | Default 587 |
| Enable SSL/TLS | Yes/No | Default yes |
| SMTP password | Secret (masked) | Required |
| From email address | Text | Valid email format |
| From display name | Text | Default "Server Backup Tool" |

Then a loop to add recipients:

```
Add a recipient:
  Email: admin@example.com
  Display name: Admin

Add another recipient? [y/n]
```

Email triggers (Open, Close, Heartbeat) are configured with default subjects and content. The installer generates sensible defaults that can be edited in App.config later.

### Step 6 — API Configuration (if selected)

| Prompt | Input Type | Validation |
|---|---|---|
| API port | Number | Default 5000, check port is available |
| Database location | Text (path) | Default `{InstallDir}\Data\sbt.db`, directory must be writable |

After collecting the inputs, the installer generates a client ID and client secret (random strings), displays them to the user, and hashes them with SHA-512 for storage in `appsettings.json`. The plaintext values are shown once and never stored.

```
┌──────────────────────────────────────────────────────────────┐
│  ⚠  API Credentials — COPY THESE NOW                        │
│                                                              │
│  Client ID:     f7a3b2c1-9e8d-4f6a-b5c3-2d1e0f9a8b7c       │
│  Client Secret: kP9mN4xR2vL8wQ5jT7yF3aB6cD0eH1iG           │
│                                                              │
│  These credentials are hashed before storage.                │
│  You will NOT be able to retrieve them from the              │
│  configuration file after installation.                      │
│                                                              │
│  Press any key once you have copied them...                  │
└──────────────────────────────────────────────────────────────┘
```

### Step 7 — Confirmation

Display a summary table of all selections before proceeding:

```
┌──────────────────────────────────────────────┐
│ Installation Summary                         │
├──────────────────┬───────────────────────────┤
│ Components       │ SBT, API                  │
│ Install location │ C:\Program Files\...      │
│ Server name      │ Minecraft-1               │
│ Game type        │ Minecraft                 │
│ Server directory │ D:\Servers\Minecraft      │
│ Backup time      │ 03:00:00                  │
│ Email            │ Enabled (smtp.gmail.com)  │
│ API port         │ 5000                      │
│ API auth         │ Client ID + Secret (hash) │
│ Database         │ ...\Data\sbt.db           │
└──────────────────┴───────────────────────────┘

Proceed with installation? [y/n]
```

### Step 8 — Installation (with progress)

```
Installing Server Backup Tool...

  [████████████████████████████████████████] 100%
  ✓ Created install directory
  ✓ Copied SBT files (24 files)
  ✓ Generated App.config
  ✓ Created Logs directory
  ✓ Created Archived Logs directory
  ✓ Created ProgramData directory
  ✓ Copied API files (18 files)
  ✓ Generated appsettings.json
  ✓ Hashed API credentials (SHA-512)
  ✓ Created SQLite database
  ✓ Enabled WAL mode
  ✓ Created Logs table
  ✓ Created Commands table
  ✓ Registered scheduled task

Installation complete.
```

Each step updates the progress bar and shows a tick or cross.

### Step 9 — Post-Install Validation

The installer runs a series of checks:

| Check | How |
|---|---|
| App.config is valid XML | Parse with XDocument |
| Server directory is accessible | Directory.Exists |
| Start file exists | File.Exists |
| SQLite database opens | Open connection, verify Logs and Commands tables exist |
| ProgramData directory is writable | Write and delete a temp file |
| Scheduled task is registered | Query Task Scheduler |

```
Post-install checks:

  [ok] Configuration file valid
  [ok] Server directory accessible
  [ok] Start file found
  [ok] Database connection successful
  [ok] ProgramData directory writable
  [ok] Scheduled task registered

All checks passed. You can now start the Server Backup Tool.
```

---

## 5. Update Flow

### Detection

The installer checks for an existing installation by looking for a known marker:

1. Check the default install location
2. Check the Windows Registry uninstall key (if registered during install)
3. Allow the user to point to an existing install directory

### Update Steps

1. Detect current version (read the existing SBT assembly version)
2. Compare against the installer's bundled version
3. If the bundled version is newer, show what's changing
4. Back up the current App.config, appsettings.json, and database
5. Replace application binaries (SBT and/or API)
6. Preserve App.config and appsettings.json — do not overwrite
7. Run database migrations if the schema has changed (add new columns/tables)
8. Verify the installation post-update

```
Current version: 2.0.2
New version:     2.1.0

The following will be updated:
  • Server Backup Tool binaries
  • SBT API binaries

The following will NOT be touched:
  • App.config (your settings)
  • appsettings.json (your API settings and credentials)
  • sbt.db (your data)
  • Logs directory

Proceed? [y/n]
```

### Rollback

If the update fails at any step:

1. Restore the backed-up binaries
2. Restore the backed-up App.config and appsettings.json
3. Restore the backed-up database
4. Report what failed and why

---

## 6. Configure Flow

Allows editing the SBT's App.config after installation without reinstalling. The installer reads the existing configuration, presents the current values, and lets the user modify individual sections.

### Detection

Uses the same detection logic as the update flow to locate the existing installation and its App.config.

### Section Selection

```
┌─────────────────────────────────────┐
│   Server Backup Tool — Configure    │
│   Installation: C:\Program Files\...│
└─────────────────────────────────────┘

Which section would you like to edit?

> Server Details
  Backup and Timers
  Email Notifications
  Database Settings
  Done
```

The user can edit multiple sections in a single session. After each section, they return to this menu. Selecting "Done" saves all changes and exits.

### Server Details

Displays the current values and allows editing:

```
Current server details:

  Server name:      Minecraft-1
  Game type:        Minecraft
  Server directory: D:\Servers\Minecraft
  Start file:       start.bat
  Server IP:        192.168.1.100

Edit a field:

> Server name
  Game type
  Server directory
  Start file
  Server IP address
  Back (no changes)
```

Each field uses the same validation as the install flow (e.g., directory must exist, valid IPv4 format). The current value is shown as the default so the user can press Enter to keep it.

### Backup and Timers

Displays the current backup time and custom timers:

```
Current timer configuration:

  Backup time: 03:00:00

  Custom timers:
    1. Warning  — 02:50:00 — "Server backup starting in 10 minutes"
    2. Final    — 02:59:00 — "Server backup starting in 1 minute"

What would you like to do?

> Change backup time
  Add a custom timer
  Edit a custom timer
  Remove a custom timer
  Back (no changes)
```

### Email Notifications

Displays the current SMTP settings and configured email triggers with their recipients:

```
Current email configuration:

  SMTP server:  smtp.gmail.com
  Port:         587
  SSL/TLS:      Enabled
  From address: server@example.com (Server Backup Tool)

  Configured triggers:
    1. Open      — admin@example.com (Admin)
    2. Close     — admin@example.com (Admin)
    3. Heartbeat — admin@example.com (Admin)

What would you like to do?

> Edit SMTP settings
  Add a recipient to a trigger
  Remove a recipient from a trigger
  Add a new trigger
  Remove a trigger
  Back (no changes)
```

**Add a recipient to a trigger:**

```
Select a trigger:

> Open
  Close
  Heartbeat

Add a recipient:
  Email: ops@example.com
  Display name: Ops Team

✓ Added ops@example.com to the Open trigger.
```

**Add a new trigger:**

Allows the user to add a trigger that fires on a specific log level or keyword in server output (matching the existing `system="false"` email triggers in App.config):

```
Trigger on: ERROR
  Email: admin@example.com
  Display name: Admin
  Subject: Server Error Detected
  Content file (or inline): Server.html

✓ Added trigger for "ERROR".
```

### Database Settings

Displays the current database path and polling interval:

```
Current database configuration:

  Database path:    C:\ProgramData\...\Data\sbt.db
  Polling interval: 1000 ms

Edit a field:

> Database path
  Polling interval
  Back (no changes)
```

### Saving Changes

When the user selects "Done" from the section menu:

1. Back up the current App.config to `App.config.bak`
2. Rebuild the App.config XML with the modified values using the same `ConfigWriter` service used during installation
3. Validate the generated XML by parsing with `XDocument`
4. Write the new App.config
5. If the SBT is currently running, prompt the user to restart it for changes to take effect

```
Changes saved to App.config.

  ✓ Backed up existing config to App.config.bak
  ✓ Updated Email Notifications — added 1 recipient
  ✓ Updated Backup and Timers — changed backup time

  The SBT is currently running. Restart now? [y/n]
```

---

## 7. Uninstall Flow

1. Confirm with the user
2. Stop the SBT if it's running (kill the process)
3. Remove the scheduled task
4. Remove the server's `Logs` and `Commands` rows (filtered by `ServerName`)
5. If no other server names remain in the database, ask whether to delete the database file entirely
6. Ask whether to keep or delete local logs
7. Remove application files
8. Remove the ProgramData directory
9. Remove the Registry uninstall entry (if present)

```
Uninstalling Server Backup Tool...

  The SBT is currently running. Stop it? [y/n]

  Keep your data?
  > Keep database and logs
    Delete everything

  ✓ Stopped SBT process
  ✓ Removed scheduled task
  ✓ Removed application files
  ✓ Removed ProgramData directory
  - Kept database (D:\...\Data\sbt.db)
  - Kept logs (D:\...\Logs\)

Uninstall complete.
```

---

## 8. SQLite Database Setup

A single shared database is used by all SBT installs on the same machine. Multiple SBT instances (e.g. one per game server) and the API all read and write to the same `sbt.db` file. Every record is tagged with a `ServerName` (the server's name from App.config) so each instance only sees its own data, and the API can filter logs and commands by server name. Only one server with a given name can run at any one time.

When the API component is selected, the installer creates the database if it does not already exist. If the database already exists (from a previous SBT install), the installer skips table creation.

```sql
-- Enable WAL mode for concurrent access
PRAGMA journal_mode=WAL;

-- Log entries written by the SBT, read by the API
CREATE TABLE IF NOT EXISTS Logs (
    Id          INTEGER PRIMARY KEY AUTOINCREMENT,
    ServerName  TEXT    NOT NULL,
    Timestamp   TEXT    NOT NULL,
    Level       TEXT    NOT NULL,
    Logger      TEXT    NOT NULL,
    Message     TEXT    NOT NULL
);

-- Commands written by the API, read by the SBT
CREATE TABLE IF NOT EXISTS Commands (
    Id          INTEGER PRIMARY KEY AUTOINCREMENT,
    ServerName  TEXT    NOT NULL,
    Target      TEXT    NOT NULL,
    Command     TEXT    NOT NULL,
    CreatedAt   TEXT    NOT NULL
);

-- Index for the API polling new log entries per server
CREATE INDEX IF NOT EXISTS IX_Logs_Server ON Logs (ServerName, Id);

-- Index for the SBT polling commands for its server
CREATE INDEX IF NOT EXISTS IX_Commands_Server ON Commands (ServerName);
```

### Install Behaviour

- **First install on the machine:** Creates the database and tables.
- **Subsequent installs:** Opens the existing database. Tables already exist so creation is skipped via `IF NOT EXISTS`.
- **Each SBT instance** uses its server name (from the `name` attribute on `serverDetails` in App.config) when writing logs and reading commands. No additional ID is needed.
- **The API** filters logs and commands by `ServerName`. The list of known servers is derived from the distinct `ServerName` values in the Logs table.

### Uninstall Behaviour

When an SBT instance is uninstalled, the installer optionally cleans up its associated `Logs` and `Commands` rows (filtered by `ServerName`). If no other server names remain in the database, the installer offers to delete the database file entirely.

---

## 9. App.config Generation

The installer builds the App.config programmatically using `XDocument`. This avoids string concatenation and ensures valid XML.

The generated file includes:

- `serverBackup` custom section with all user-provided values
- `log4net` section with rolling file appenders (using forward slashes for cross-platform compatibility)
- Default email triggers (Open, Close, Heartbeat) with sensible subject lines

The installer does not modify App.config during updates.

When the API component is selected, the installer also generates `appsettings.json` for the API project. This includes the database path, webhook URL (if provided), and the SHA-512 hashed client ID and secret. The installer does not modify `appsettings.json` during updates — if the user needs to regenerate credentials, they must reinstall the API component or manually update the hashes.

---

## 10. Scheduled Task Registration

On Windows, the installer registers the SBT with Task Scheduler:

| Setting | Value |
|---|---|
| Name | Server Backup Tool |
| Trigger | At system startup |
| Action | Run `Server Backup Tool.exe` |
| Working directory | The install directory |
| Run as | The current user (or a specified service account) |
| Run whether user is logged on | Yes (requires password prompt) |
| Restart on failure | Every 1 minute, up to 3 times |

On Linux (future), this would register a systemd service instead.

---

## 11. Registry Entries (Windows)

To integrate with Add/Remove Programs, the installer writes to:

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\ServerBackupTool
```

| Value | Data |
|---|---|
| DisplayName | Server Backup Tool |
| DisplayVersion | 2.0.2 |
| Publisher | Hunter Industries |
| InstallLocation | C:\Program Files\Hunter Industries\Server Backup Tool |
| UninstallString | "C:\...\SBTInstaller.exe" --uninstall |
| DisplayIcon | C:\...\Content\Logo.ico |
| NoModify | 1 |
| NoRepair | 1 |

---

## 12. Distribution

The installer is published as a **self-contained single-file executable**:

```bash
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true
```

This produces a single `SBTInstaller.exe` that includes the .NET runtime, all dependencies, and the pre-published SBT and API binaries as embedded resources.

The published SBT and API files are embedded in the installer at build time so the installer is fully self-contained — no internet connection required during installation.

---

## 13. Build Order

| Phase | Work | Depends On |
|---|---|---|
| 1 | Project scaffolding — create the installer project, add Spectre.Console, set up the mode detection and step pipeline | Nothing |
| 2 | Install location step — directory selection, creation, permission validation | Phase 1 |
| 3 | Server configuration step — collect server details, validate paths | Phase 1 |
| 4 | Timer configuration step — backup time and custom timers | Phase 1 |
| 5 | Email configuration step — SMTP settings and recipients | Phase 1 |
| 6 | App.config generation — build XML from collected inputs | Phases 3, 4, 5 |
| 7 | File deployment step — copy published SBT files to install directory, create runtime directories | Phase 2 |
| 8 | Component selection step — optional API checkbox | Phase 1 |
| 9 | API configuration step — port and database path | Phase 8 |
| 10 | Database setup step — create SQLite DB, WAL, tables, indexes | Phase 9 |
| 11 | Scheduled task registration | Phase 7 |
| 12 | Registry entries for Add/Remove Programs | Phase 7 |
| 13 | Post-install validation checks | Phases 6, 7, 10, 11 |
| 14 | Confirmation summary and progress display | All install phases |
| 15 | Update mode — version detection, backup, binary replacement, migration | Phase 14 |
| 16 | Configure mode — config reader, section selection menu, field editing, App.config rewrite with backup | Phase 6, 14 |
| 17 | Uninstall mode — cleanup, task removal, registry removal | Phase 14 |
| 18 | Embedded resources — bundle published SBT/API into the installer | Phase 14 |
| 19 | Single-file publish and testing | Phase 18 |
