# Linux Compatibility Plan

This document assesses which projects are compatible with Linux and outlines the work required to make Windows-only projects cross-platform.

---

## Compatibility Summary

| Project | Status | Effort |
|---|---|---|
| PythonApps | Compatible | None |
| Portfolio | Compatible | Minimal config change |
| ServerStatus | Mostly Compatible | Minor fixes |
| GitHubScraper | Mostly Compatible | Minor fixes |
| Hunter-Industries-API | Not Compatible | Major rewrite (API backend) |
| DeploymentManager | Not Compatible | Major rewrite |
| GoogleDriveSync | Not Compatible | Full rewrite |
| Server-Backup-Tool | Not Compatible | Moderate rewrite |
| NASA Image Report | Not Compatible | Moderate rewrite |
| Github to Codecks | Not Compatible | Full rewrite |

---

## Should These Projects Be Made Linux-Compatible?

Not every project benefits from Linux compatibility. Some are Windows-specific by design — they manage Windows infrastructure, depend on Windows-only software, or target Windows desktop users. Porting them would mean replacing the core purpose of the application, not just swapping out a few libraries.

### Worth porting

| Project | Reason |
|---|---|
| **Portfolio** | A web app (React + Express) with no inherent OS dependency. Already runs anywhere Node.js does — just needs a config path change. |
| **ServerStatus** | A monitoring suite (Blazor Server + console apps) targeting .NET 8 with a cross-platform TFM. The code is already nearly there; only minor path and config fixes are needed. Running on Linux aligns with hosting it on the Pi 5 or GMKtec. |
| **GitHubScraper** | A data pipeline (console app) targeting .NET 8 with no UI or OS-specific logic. Just needs `Path.Combine` fixes. Could easily run as a cron job on Linux. |
| **PythonApps** | Already fully compatible. Pure Python with cross-platform libraries. |

### Not worth porting

| Project | Reason |
|---|---|
| **DeploymentManager** | Exists specifically to deploy to Windows servers by managing IIS sites, app pools, and Windows Scheduled Tasks on remote machines. The entire purpose of the application is Windows infrastructure management. A Linux version would be a completely different tool. |
| **Hunter-Industries-API** | Built on classic ASP.NET Web API 2 / .NET Framework 4.7.2 hosted on IIS. While the API concept itself is platform-neutral, the effort to rewrite 20 controllers from `System.Web` to ASP.NET Core is substantial and only worthwhile if there is a concrete plan to host on Linux. Currently deployed on Windows with IIS. |
| **GoogleDriveSync** | A WinForms desktop app for syncing files between a local Windows directory and Google Drive. Uses NTFS-specific features (Alternate Data Streams, Hidden file attributes) and Windows drive letter detection as core functionality. The Google Drive API logic is portable, but the UI and filesystem layer would need a full rewrite. |
| **Server-Backup-Tool** | Designed to supervise game servers on Windows — launches `.bat` files, detects Windows CMD `PAUSE` prompts, and manages processes in a Windows-specific way. The game servers it manages (e.g. Minecraft) are run on Windows with batch scripts. Porting this only makes sense if the game servers themselves move to Linux. |
| **NASA Image Report** | Depends on Microsoft Word being installed to generate `.docx` reports via COM interop. Replacing this with Open XML SDK is technically possible, but the project is a small utility — the rewrite effort outweighs the benefit unless there is a specific need to run it on Linux. |
| **Github to Codecks** | A small WinForms wizard for migrating GitHub Issues to Codecks. Only used occasionally as a manual utility. The WinForms UI and .NET Framework 4.7.2 runtime make it Windows-only, and its infrequent use does not justify a cross-platform rewrite. |

---

## Already Compatible

### PythonApps (BirthdayNotifier, ForumHelperBot, DNS Checker)

- **Language:** Python 3.12+
- **Dependencies:** `discord.py`, `aiohttp`, `requests`, `python-dotenv` — all cross-platform.
- **Windows-specific code:** None.
- **Work required:** None. Add a `requirements.txt` to each sub-project to formalise dependencies.

### Portfolio (React frontend + Express proxy)

- **Language:** TypeScript (React 19 + Express 5)
- **Dependencies:** All standard npm packages, fully cross-platform.
- **Windows-specific code:** None in source.
- **Work required:**
  - Change the default `MEDIA_PATH` in the `.env` from `C:/inetpub/wwwroot` to a Linux-appropriate path (e.g. `/var/www/media`) when deploying on Linux. This is a config-only change; the code already reads from the environment variable.
  - Optionally remove the `--use-system-ca` Node flag from the dev start script if not needed on Linux.

---

## Mostly Compatible (Minor Fixes)

### ServerStatus (.NET 8, Blazor Server + Console apps)

- **Framework:** .NET 8.0 (`net8.0`) — cross-platform TFM.
- **Issues found:**
  1. **log4net file paths use backslashes** — `Logs\SSR.log`, `Logs\SSA.log`, `Logs\Site.log` in config files. On Linux, `\` is treated as a literal character, not a directory separator. Log files would be created with wrong names.
  2. **Unused `System.Management` NuGet reference** in `ServerStatusReporter.csproj` — Windows-only package, but not used in any source file.
  3. **Test config paths** use `@"Mocks\Configs\..."` backslash literals — tests would fail on Linux.
  4. **CI/CD** runs on `windows-latest` only.
- **Work required:**
  1. Change log4net config paths from `Logs\SSR.log` to `Logs/SSR.log` (or use forward slashes).
  2. Remove the unused `System.Management` package reference from `ServerStatusReporter.csproj`.
  3. Update test path strings to use `Path.Combine("Mocks", "Configs", "Test.config")` instead of hardcoded backslashes.
  4. Update GitHub Actions workflows to use `ubuntu-latest` (or a matrix of both).
  5. Remove the IIS Express profile from `launchSettings.json` (Kestrel profiles already exist).

### GitHubScraper (.NET 8, Console app)

- **Framework:** .NET 8.0 (`net8.0`) — cross-platform TFM.
- **Issues found:**
  1. **Hardcoded backslash path separators** in `Database Service.cs` — 9 occurrences of `$@"{_Options.SQLFiles}\filename.sql"`. These would produce invalid paths on Linux.
  2. **Unused `System.Data.SqlClient`** legacy package alongside the modern `Microsoft.Data.SqlClient`.
  3. **Test data** uses `@"C:\SQL"` — test-only, but would look odd on Linux.
  4. **CI/CD** runs on `windows-latest` only.
- **Work required:**
  1. Replace all `$@"{_Options.SQLFiles}\filename.sql"` with `Path.Combine(_Options.SQLFiles, "filename.sql")` in `Database Service.cs`.
  2. Remove the unused `System.Data.SqlClient` package reference.
  3. Update test path stubs to use platform-agnostic values.
  4. Update GitHub Actions workflows to use `ubuntu-latest` (or a matrix of both).
  5. Change the log4net config path from `Logs\Scraper.log` to `Logs/Scraper.log`.

---

## Not Compatible (Significant Work Required)

### Hunter-Industries-API (.NET Framework 4.7.2 API + .NET 10 Control Panel)

- **Framework:** API is .NET Framework 4.7.2 with `System.Web`/OWIN on IIS. Control Panel is .NET 10 Blazor Server.
- **Blocking dependencies:**
  - `System.Web` and `System.Web.Http` (classic ASP.NET Web API 2) — Windows/IIS only, no Linux equivalent.
  - `Microsoft.Owin.Host.SystemWeb` — IIS pipeline integration, Windows only.
  - Hardcoded absolute Windows paths in `Web.config` (`C:\Users\TobyH\...`) and `appsettings.json`.
  - All 20 controllers built on `System.Web.Http.ApiController`.
  - CI/CD targets `windows-latest` with `msbuild`/`nuget restore`.
- **Work required:**
  1. **Rewrite the API project from .NET Framework 4.7.2 to ASP.NET Core (.NET 8 or .NET 10).** This is the largest single piece of work across all projects. Every controller must be migrated from `System.Web.Http.ApiController` to `Microsoft.AspNetCore.Mvc.ControllerBase`. All `System.Web.HttpContext` usage must be replaced with ASP.NET Core's `HttpContext`. The OWIN pipeline must be replaced with ASP.NET Core middleware. Swagger must be migrated from Swashbuckle.Core (classic) to Swashbuckle.AspNetCore or an alternative.
  2. Replace `System.Data.SqlClient` with `Microsoft.Data.SqlClient` in the API project.
  3. Replace hardcoded absolute Windows paths in config files with relative paths or environment variables.
  4. Replace `Web.config` with `appsettings.json` configuration.
  5. Update the Control Panel's `AuthPayloadLocation` config to use a relative or environment-variable-based path.
  6. Update CI/CD workflows to build with `dotnet build` and run on `ubuntu-latest`.
  7. The Common library (`netstandard2.0` + `net10.0`) and Control Panel (`net10.0`) are already cross-platform or nearly so — they just need the config path fixes.

### DeploymentManager (.NET 10, Blazor Server)

- **Framework:** .NET 10.0 Blazor Server — the web framework itself is cross-platform.
- **Blocking dependencies:**
  - `Microsoft.Web.Administration` — manages IIS sites/app pools. No Linux equivalent.
  - `TaskScheduler` (Microsoft.Win32.TaskScheduler) — manages Windows Task Scheduler. No Linux equivalent.
  - P/Invoke into `advapi32.dll` (`LogonUser`) and `kernel32.dll` (`CloseHandle`) for Windows impersonation.
  - `WindowsIdentity.RunImpersonated()` — Windows security API.
  - All deployment paths built with Windows drive letters (`$"{drive}:"`).
- **Work required:**
  1. **Abstract the service management layer** behind an interface. On Windows, keep the existing IIS (`Microsoft.Web.Administration`) and Task Scheduler (`Microsoft.Win32.TaskScheduler`) implementations. On Linux, create new implementations using `systemctl` (for systemd services) or equivalent process management.
  2. **Replace P/Invoke impersonation** with SSH-based remote execution on Linux (e.g. `SSH.NET` NuGet package). The current approach of impersonating a Windows domain user to manage remote IIS/tasks has no Linux analogue — remote Linux management would typically use SSH keys.
  3. **Rework the path model** to support Linux paths (no drive letters). The `$"{drive}:"` pattern must be replaced with a configurable root path per environment.
  4. **Create platform-conditional service registration** using `RuntimeInformation.IsOSPlatform()` to wire up the correct implementations at startup.
  5. Update CI/CD to support Linux runners.

### GoogleDriveSync (.NET Framework 4.7.2, WinForms)

- **Framework:** .NET Framework 4.7.2, WinForms desktop app.
- **Blocking dependencies:**
  - WinForms UI (`System.Windows.Forms`) — Windows only.
  - `System.Drawing` (GDI+) — Windows only.
  - NTFS Alternate Data Streams (`Zone.Identifier` removal) — Windows filesystem feature.
  - `FileAttributes.Hidden` — Windows filesystem attribute.
  - Windows drive letter detection (`":\\"` checks in flow control).
  - Hardcoded backslash path separators throughout all path construction.
  - `System.Deployment` (ClickOnce), `System.Management` (WMI) references.
  - All NuGet packages pinned to `net462` builds.
- **Work required:**
  1. **Migrate from .NET Framework 4.7.2 to .NET 8+.**
  2. **Replace WinForms UI with a cross-platform alternative.** Options:
     - **Avalonia UI** — most mature cross-platform .NET desktop UI framework, closest to WPF/WinForms in workflow.
     - **MAUI** — Microsoft's cross-platform framework (Windows, macOS, iOS, Android; Linux support is community-driven).
     - **Blazor Hybrid** — if a web-style UI is acceptable.
     - **Terminal UI (Spectre.Console)** — if a GUI is not essential.
  3. Replace all hardcoded `\` path separators with `Path.Combine()` or `Path.DirectorySeparatorChar`.
  4. Remove or conditionalise the NTFS-specific code (`Zone.Identifier` ADS removal, `FileAttributes.Hidden`).
  5. Replace drive letter detection logic with a platform-agnostic path model.
  6. Remove unused references (`System.Deployment`, `System.Management`).
  7. Migrate Google API packages from `net462` to `netstandard2.0` / `net8.0` builds.
  8. Update `packages.config` to SDK-style `PackageReference`.

### Server-Backup-Tool (.NET 6, Console app)

- **Framework:** .NET 6.0 (`net6.0`) — cross-platform TFM.
- **Blocking dependencies:**
  - Server start file assumed to be a `.bat` (Windows batch) file.
  - Minecraft shutdown detection looks for Windows CMD `PAUSE` prompt (`{workingDirectory}>PAUSE`).
  - Hardcoded backslash path separators in `Job Converter.cs` and `Job Service.cs`.
  - ICMP ping requires elevated privileges on Linux (root or `cap_net_raw`).
  - CI/CD uses `windows-latest` with PowerShell scripting.
- **Work required:**
  1. **Support shell scripts (`.sh`) as server start files** alongside `.bat`. Add a config option or auto-detect based on OS (`RuntimeInformation.IsOSPlatform()`).
  2. **Rework the Minecraft final-message detection** to handle both Windows CMD output (`{path}>PAUSE`) and Linux shell output (e.g. detect process exit or a configurable shutdown string).
  3. Replace all hardcoded `\` path separators with `Path.Combine()`.
  4. Document that ICMP ping on Linux requires either running as root or granting `cap_net_raw` capability (`sudo setcap cap_net_raw+ep ./ServerBackupTool`).
  5. Update CI/CD to support Linux runners.
  6. Consider upgrading from .NET 6 (end of support) to .NET 8 LTS.

### NASA Image Report (.NET 6, Console app)

- **Framework:** .NET 6.0 (`net6.0`) — cross-platform TFM.
- **Blocking dependencies:**
  - `Microsoft.Office.Interop.Word` (COM interop) — requires Microsoft Word installed on Windows. No Linux equivalent.
  - Hardcoded backslash path separators in 4 locations.
  - `System.Net.WebClient` (deprecated but cross-platform).
- **Work required:**
  1. **Replace COM Word interop with a cross-platform document generation library.** Options:
     - **Open XML SDK (`DocumentFormat.OpenXml`)** — Microsoft's official library for creating `.docx` files without Word installed. Free, cross-platform, full control over document structure.
     - **QuestPDF** — if PDF output is acceptable instead of Word format.
     - **NPOI** — can generate `.docx` files, cross-platform.
  2. Replace all hardcoded `\` path separators with `Path.Combine()`.
  3. Replace deprecated `WebClient` with `HttpClient`.
  4. Consider upgrading from .NET 6 (end of support) to .NET 8 LTS.

### Github to Codecks (.NET Framework 4.7.2, WinForms)

- **Framework:** .NET Framework 4.7.2, WinForms desktop app.
- **Blocking dependencies:**
  - WinForms UI (`System.Windows.Forms`) — Windows only.
  - `System.Drawing` (GDI+) — Windows only.
  - `System.Deployment` (ClickOnce) — Windows only.
  - .NET Framework 4.7.2 — Windows only runtime.
  - Hardcoded `D:\` publish path.
  - Legacy `packages.config` NuGet format.
- **Work required:**
  1. **Migrate from .NET Framework 4.7.2 to .NET 8+.**
  2. **Replace WinForms with a cross-platform alternative** (same options as GoogleDriveSync above). However, given this is a simple wizard-style utility (4 dropdowns and a button), a simpler approach may be better:
     - **Console/CLI app** — the workflow is linear (select repo, select issue, select project, confirm). A CLI with `Spectre.Console` for rich prompts would be a natural fit and inherently cross-platform.
     - **Avalonia UI** — if a GUI is preferred.
  3. Migrate from `packages.config` to SDK-style `PackageReference`.
  4. Remove `System.Deployment` reference.
  5. The HTTP service logic (`GithubService`, `CodeckService`) uses RestSharp and is already platform-neutral — it can be reused directly.

---

## Recommended Priority Order

Based on effort vs. impact:

1. **Portfolio** — config-only change, immediate benefit.
2. **ServerStatus** — minor code fixes, the monitoring suite becomes Linux-ready.
3. **GitHubScraper** — minor code fixes, the data pipeline becomes Linux-ready.
4. **NASA Image Report** — moderate effort, replace Word COM with Open XML SDK.
5. **Server-Backup-Tool** — moderate effort, mostly adding Linux-aware process management.
6. **Github to Codecks** — rewrite as a CLI tool with Spectre.Console, small codebase.
7. **Hunter-Industries-API** — large effort, but the API is the backbone of the portfolio ecosystem.
8. **GoogleDriveSync** — full UI rewrite needed, large effort.
9. **DeploymentManager** — fundamentally tied to Windows infrastructure (IIS, Task Scheduler), cross-platform only makes sense if deploying to Linux targets.

---

## Common Patterns Across Projects

Several issues recur across multiple projects and can be addressed systematically:

### Backslash Path Separators
**Affected:** GitHubScraper, ServerStatus, Server-Backup-Tool, NASA Image Report, GoogleDriveSync.
**Fix:** Replace `$@"{path}\file"` with `Path.Combine(path, "file")` throughout. Use `Path.DirectorySeparatorChar` where a separator literal is needed.

### .NET Framework 4.7.2 to .NET 8+ Migration
**Affected:** Hunter-Industries-API, GoogleDriveSync, Github to Codecks.
**Fix:** Create new SDK-style `.csproj` files targeting `net8.0`, migrate NuGet references from `packages.config` to `PackageReference`, and resolve any API differences.

### CI/CD Windows-Only Runners
**Affected:** GitHubScraper, ServerStatus, Server-Backup-Tool, DeploymentManager, GoogleDriveSync, Hunter-Industries-API.
**Fix:** Change `runs-on: windows-latest` to `ubuntu-latest`, or use a matrix strategy to test on both. Replace any PowerShell-specific CI scripts with cross-platform alternatives.

### log4net Config Backslashes
**Affected:** GitHubScraper, ServerStatus, Server-Backup-Tool.
**Fix:** Change `Logs\filename.log` to `Logs/filename.log` in all log4net XML config. Forward slashes work on both Windows and Linux.
