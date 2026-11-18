# Archive Scanner — Project Plan

A Python utility that monitors AO3 (Archive of Our Own) for new fanfiction by relationship tag, including works restricted to logged-in users, and sends digest email notifications. Runs on a schedule via Windows Task Scheduler.

---

## 1. Context

AO3's built-in RSS feeds for relationship tags exclude restricted (logged-in-only) works. AO3 has no native tag subscription with email notifications — only work, series, and author subscriptions send emails. This project fills that gap: an authenticated poller that searches AO3 by relationship tag, sees restricted works, and emails a digest of anything new.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────┐
│              Windows Task Scheduler                  │
│         (triggers every hour via .bat file)          │
└──────────────────────┬──────────────────────────────┘
                       │ runs
┌──────────────────────▼──────────────────────────────┐
│                archive_scanner.py                    │
│                                                     │
│  1. Load config (config.json)                       │
│  2. Login to AO3 (ao3.py library, async)            │
│  3. For each tracked relationship tag:              │
│     a. Search AO3 sorted by date posted             │
│     b. Compare results against seen_works.json      │
│     c. Collect new work metadata                    │
│  4. If new works found:                             │
│     a. Build HTML digest email grouped by ship      │
│     b. Send via configured SMTP                     │
│  5. Update seen_works.json with new IDs             │
│  6. Log run to scanner.log                          │
│                                                     │
└─────────────────────────────────────────────────────┘

Data files (in app directory):
  config.json        — AO3 credentials, SMTP settings, tracked ships
  seen_works.json    — previously seen work IDs per relationship
  scanner.log        — rotating log file
```

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Language | Python 3.12+ | User has Python experience, rich library ecosystem |
| AO3 Client | ao3.py (`pip install ao3.py`) | Async, typed, supports login + WorkSearchOptions with `relationship_names` filter, paginated search |
| Fimfiction Client | requests + OAuth2 | Official REST API (v2), JSON:API spec, character tag search |
| XenForo (SB/SV) | feedparser | RSS feed parsing for SpaceBattles and Sufficient Velocity forum listings |
| FFNet (experimental) | curl_cffi | Cloudflare bypass via TLS fingerprint impersonation. Fragile — may break when Cloudflare updates |
| Wattpad (experimental) | requests | Undocumented internal API endpoints. May break without notice |
| Quotev (experimental) | requests + BeautifulSoup | Web scraping of tag listing pages |
| HTTP | aiohttp (ao3.py dependency) | Async HTTP client used internally by ao3.py |
| Email | smtplib + email.mime (stdlib) | No external dependency needed, works with any SMTP provider |
| Email Templates | string.Template (stdlib) | Lightweight HTML templating without pulling in Jinja2 |
| State | JSON file or SQLite | JSON for standalone use (default), SQLite for shared state with Bibliotheca Fabulae |
| Logging | logging (stdlib) | RotatingFileHandler for log rotation |
| Scheduling | Windows Task Scheduler | Native to Windows, no daemon process needed |

### Site adapter architecture

Each supported site implements a common adapter interface:

```python
class SiteAdapter:
    async def authenticate(config) -> bool
    async def search(tag, max_pages) -> list[WorkResult]
    async def validate_tag(tag) -> bool
    def get_tag_format_help() -> str
```

| Adapter | Tier | Search Capability | Tag Format |
|---------|------|-------------------|------------|
| AO3 | Stable | Relationship tags | Canonical: `Character A/Character B` |
| Fimfiction | Stable | Character tags | Character IDs: `#fluttershy #rainbow-dash` |
| SpaceBattles | Stable | Fandom/character tags (no pairings) | Forum tags: `worm`, `taylor-hebert` |
| Sufficient Velocity | Stable | Fandom/character tags (no pairings) | Forum tags (same as SpaceBattles) |
| Wattpad | Experimental | Freeform ship-name tags | Concatenated: `drarry`, `bakudeku` |
| Quotev | Experimental | Freeform ship-name tags | Freeform: `drarry`, `draco-x-harry` |
| FFNet | Experimental | Character pair filter | Numeric character IDs + pairing checkbox |

**Experimental adapters** may break when sites update their anti-bot measures or internal APIs. They log warnings when access fails and skip gracefully rather than crashing the scan run.

### Why ao3.py over alternatives

| Library | PyPI Package | Status | Auth | Relationship Search | Notes |
|---------|-------------|--------|------|-------------------|-------|
| **ao3.py** | `ao3.py` | Active (docs at ao3py.readthedocs.io) | Yes — `client.login()` | Yes — `WorkSearchOptions(relationship_names=[...])` | Async, typed, paginated search, best API surface |
| ao3-api | `ao3-api` | v2.3.1 (Jan 2025) | Yes — `AO3.Session()` | Via general `Search()` | Sync, has rate limit helper, less actively updated |
| ao3-py | `ao3-py` | v2026.10.10 | Yes | Less documented search API | Sync, httpx-based, lazy objects |

ao3.py is chosen for its explicit `relationship_names` filter in `WorkSearchOptions`, paginated async search via `work_search_pages()`, and active documentation.

---

## 4. Configuration — `config.json`

```json
{
  "ao3": {
    "username": "your_ao3_username",
    "password": "your_ao3_password"
  },
  "smtp": {
    "host": "smtp.example.com",
    "port": 587,
    "username": "your_email@example.com",
    "password": "your_email_password",
    "use_tls": true
  },
  "email": {
    "from_address": "Archive Scanner <your_email@example.com>",
    "to_address": "you@example.com",
    "subject_prefix": "[Archive Scanner]"
  },
  "relationships": [
    {
      "site": "ao3",
      "tag": "Character A/Character B",
      "track_from": "2026-11-18T10:30:00Z"
    },
    {
      "site": "ao3",
      "tag": "Character C & Character D",
      "track_from": "2026-11-01T00:00:00Z"
    }
  ],
  "storage": {
    "mode": "json",
    "sqlite_path": ""
  },
  "search": {
    "max_pages": 2,
    "delay_between_requests_seconds": 3
  }
}
```

| Field | Purpose |
|-------|---------|
| `ao3.username` / `ao3.password` | AO3 login credentials for accessing restricted works |
| `smtp.*` | Configurable SMTP connection — works with Gmail, Outlook, SendGrid, self-hosted, etc. |
| `smtp.use_tls` | Whether to use STARTTLS (port 587) or direct SSL (port 465 — handled automatically) |
| `email.from_address` | Sender address for digest emails |
| `email.to_address` | Recipient address (your inbox) |
| `email.subject_prefix` | Prefix for email subject lines |
| `relationships[].site` | Source site for this tag (e.g., `"ao3"`, `"fimfiction"`) |
| `relationships[].tag` | Relationship/character tag to monitor (exact tag name as it appears on the source site) |
| `relationships[].track_from` | ISO 8601 timestamp — only notify on works posted after this date. Set by the user when adding a tag. This is how the scanner avoids flooding you with existing works on first run |
| `storage.mode` | `"json"` (default, standalone) or `"sqlite"` (shared database with Bibliotheca Fabulae) |
| `storage.sqlite_path` | Path to the Bibliotheca Fabulae SQLite database file. Only used when mode is `"sqlite"`. If the path is invalid or the file doesn't exist, falls back to JSON with a warning |
| `search.max_pages` | How many pages of results to check per relationship (each page = 20 works) |
| `search.delay_between_requests_seconds` | Delay between requests to avoid rate limiting |

---

## 5. State File — `seen_works.json`

```json
{
  "ao3:Character A/Character B": [12345678, 12345600, 12345590],
  "ao3:Character C & Character D": [12345670, 12345655]
}
```

Keys are prefixed with the source site to avoid collisions across sites. Each entry maps to a list of work IDs that have already been seen and notified on. On each run, new work IDs are appended. The file is capped at 500 IDs per relationship to prevent unbounded growth — oldest IDs are trimmed when the cap is exceeded.

When `storage.mode` is `"sqlite"`, the scanner reads/writes to the Bibliotheca Fabulae database instead of this file. The same data is stored in the `SeenWorks` table, keyed by `ScannerTags`.

---

## 6. Core Logic — Polling Flow

```
1. LOAD config.json
   │
2. INITIALISE site adapters for each unique site in config.relationships
   │  (AO3: create ao3.py Client, LOGIN with credentials)
   │  (Other sites: initialise appropriate adapter with auth)
   │
3. LOAD state (seen_works.json or SQLite, based on storage.mode)
   │
4. FOR EACH relationship in config.relationships:
   │
   ├── SELECT site adapter based on relationship.site
   │
   ├── SEARCH via adapter (site-specific implementation):
   │     AO3: WorkSearchOptions with relationship_names, sorted by revised_at
   │     Other sites: equivalent tag/character search via adapter
   │     (up to max_pages pages)
   │     (delay between pages per config)
   │
   ├── FOR EACH work in results:
   │     ├── IF work.id NOT IN seen_works[relationship]
   │     │   AND work.date_posted > relationship.track_from:
   │     │     COLLECT: id, title, authors, summary,
   │     │             url, relationships, date_published,
   │     │             nwords, nchapters, is_complete,
   │     │             rating, is_restricted
   │     │     ADD to new_works list
   │     ├── ELSE IF work.id NOT IN seen_works[relationship]
   │     │   AND work.date_posted <= relationship.track_from:
   │     │     ADD work.id to seen_works (mark as seen, no notification)
   │     └── ELSE: SKIP (already seen)
   │
   ├── DELAY between relationships
   │
5. IF new_works is not empty:
   │  ├── BUILD HTML digest email
   │  │     grouped by relationship
   │  │     with work metadata and direct links
   │  ├── SEND via SMTP
   │  ├── IF storage.mode == "sqlite":
   │  │     WRITE discoveries to ScannerDiscoveries table
   │  └── LOG: "Sent digest with N new works"
   │
6. UPDATE state with all new IDs
   │  (trim to 500 per relationship if JSON mode)
   │
7. LOG: run summary
```

### Rate Limiting Strategy

AO3 rate-limits aggressively. The script mitigates this by:
- Configurable delay between requests (default 3 seconds)
- Limiting search depth (default 2 pages = 40 works per relationship)
- Running only once per hour via Task Scheduler
- Catching HTTP 429 responses and logging them rather than retrying immediately

---

## 7. Email Digest Format

HTML email with inline CSS (for email client compatibility):

```
Subject: [Archive Scanner] 5 new works found

Body:
┌──────────────────────────────────────────────────┐
│  ARCHIVE SCANNER                                  │
│  5 new works found · 17 Oct 2026, 14:00          │
│                                                   │
│  ── Character A/Character B (3 new) ───────────  │
│                                                   │
│  📖 Work Title Here                              │
│  by AuthorName · 15,234 words · 3/? chapters     │
│  Rating: Teen · Published: 16 Oct 2026           │
│  🔒 Restricted                                   │
│  Summary text here, truncated to ~200 chars...   │
│  → https://archiveofourown.org/works/12345678    │
│                                                   │
│  📖 Another Work Title                           │
│  by DifferentAuthor · 2,100 words · 1/1 chapter  │
│  Rating: General · Published: 16 Oct 2026        │
│  Complete ✓                                       │
│  Summary text here...                            │
│  → https://archiveofourown.org/works/12345679    │
│                                                   │
│  ── Character C & Character D (2 new) ─────────  │
│                                                   │
│  📖 Third Work Title                             │
│  ...                                             │
│                                                   │
└──────────────────────────────────────────────────┘
```

The restricted indicator is included so you know which works are only visible because you're logged in — these are the ones the RSS feed would have missed.

---

## 8. Project Structure

```
archive-scanner/
├── archive_scanner/
│   ├── __init__.py
│   ├── main.py              # Entry point — orchestrates the full run
│   ├── config.py             # Load and validate config.json
│   ├── email_builder.py      # Builds HTML digest email from new works
│   ├── email_sender.py       # SMTP send logic
│   ├── state.py              # Load/save state (JSON or SQLite based on config)
│   ├── state_json.py         # JSON file backend for seen_works
│   ├── state_sqlite.py       # SQLite backend for Bibliotheca Fabulae integration
│   └── adapters/
│       ├── __init__.py
│       ├── base.py           # Abstract SiteAdapter interface
│       ├── ao3.py            # AO3 adapter (ao3.py library)
│       ├── fimfiction.py     # Fimfiction adapter (official REST API)
│       ├── xenforo.py        # SpaceBattles + Sufficient Velocity (RSS feeds)
│       ├── wattpad.py        # Wattpad adapter (experimental, undocumented API)
│       ├── quotev.py         # Quotev adapter (experimental, web scraping)
│       └── ffnet.py          # FFNet adapter (experimental, curl_cffi)
├── config.json               # User configuration (gitignored)
├── config.example.json       # Example config checked into repo
├── seen_works.json           # State file — JSON mode only (gitignored)
├── scanner.log               # Log file (gitignored)
├── requirements.txt          # Python dependencies
├── run.bat                   # Batch file for Task Scheduler
├── setup_scheduler.bat       # One-time script to create the scheduled task
├── .gitignore
└── README.md
```

---

## 9. Key Files

### `main.py` — Entry Point

```python
async def main():
    config = load_config()
    state = load_state(config.storage)  # JSON or SQLite based on config
    
    # Initialise adapters for each unique site in tracked relationships
    adapters = {}
    for rel in config.relationships:
        if rel.site not in adapters:
            adapters[rel.site] = create_adapter(rel.site, config)
            await adapters[rel.site].authenticate()
    
    all_new_works = {}
    
    for rel in config.relationships:
        adapter = adapters[rel.site]
        key = f"{rel.site}:{rel.tag}"
        seen_ids = state.get_seen(key)
        
        new_works = []
        results = await adapter.search(rel.tag, config.search.max_pages)
        
        for work in results:
            if work.id not in seen_ids and work.date_posted > rel.track_from:
                new_works.append(work)
            elif work.id not in seen_ids:
                state.mark_seen(key, work.id)  # Pre-track_from, mark seen silently
        
        if new_works:
            all_new_works[key] = new_works
        
        await asyncio.sleep(config.search.delay_between_requests_seconds)
    
    if all_new_works:
        html = build_digest_email(all_new_works)
        send_email(config, html, total_count(all_new_works))
        if config.storage.mode == "sqlite":
            write_discoveries(config.storage.sqlite_path, all_new_works)
    
    state.save(all_new_works)

if __name__ == "__main__":
    asyncio.run(main())
```

### `run.bat` — Task Scheduler Entry

```bat
@echo off
cd /d "%~dp0"
python -m archive_scanner.main
```

### `setup_scheduler.bat` — One-Time Scheduler Setup

```bat
@echo off
schtasks /create ^
    /tn "Archive Scanner" ^
    /tr "\"%~dp0run.bat\"" ^
    /sc hourly ^
    /mo 1 ^
    /f
echo Task "Archive Scanner" created to run every hour.
pause
```

---

## 10. Dependencies — `requirements.txt`

```
ao3.py
feedparser
curl_cffi
beautifulsoup4
```

ao3.py pulls in aiohttp as a transitive dependency. feedparser is used for XenForo RSS (SpaceBattles/SV). curl_cffi is used for FFNet Cloudflare bypass. beautifulsoup4 is used for Quotev scraping. smtplib, email, json, sqlite3, logging, and asyncio are all stdlib.

---

## 11. Error Handling

| Scenario | Behaviour |
|----------|-----------|
| Site adapter login failure | Log error for that site, skip its relationships, continue with other sites |
| Site rate limited (HTTP 429) | Log warning, skip remaining searches for that site this run, send digest of whatever was found so far |
| Site request timeout | Log warning, skip that relationship, continue with others |
| Experimental adapter failure (Cloudflare block, API change) | Log warning with "experimental adapter" note, skip that site, continue with stable sites |
| SMTP connection failure | Log error, exit with code 1. State is NOT updated so the same works will be retried next run |
| config.json missing | Log error, exit with code 1 |
| config.json invalid | Log error with details, exit with code 1 |
| config.json has relationship with missing `track_from` | Log error, skip that relationship with clear message explaining `track_from` is required |
| seen_works.json missing (JSON mode) | Create empty state — `track_from` dates prevent false notifications for existing works |
| seen_works.json corrupted (JSON mode) | Log warning, reset to empty state — `track_from` dates prevent false notifications |
| SQLite database not found (SQLite mode) | Log error, fall back to JSON mode with warning |
| SQLite database locked (SQLite mode) | Log warning, retry once after 5 seconds, skip if still locked |
| No new works found | Log info, exit cleanly. No email sent |

The critical design choice: **state is only updated after a successful email send**. This ensures that if the email fails, the same works will be picked up on the next run.

---

## 12. Logging

Uses Python's `logging` module with a `RotatingFileHandler`:
- Log file: `scanner.log` in the project directory
- Max size: 1 MB per file, 3 backup files retained
- Format: `2026-10-17 14:00:00 INFO main: Found 3 new works for Character A/Character B`

Log levels:
- **INFO**: Run start/end, works found per ship, email sent
- **WARNING**: Rate limited, request timeout, state file issues
- **ERROR**: Login failure, SMTP failure, config errors

---

## 13. Implementation Phases

### Phase 1: Foundation
1. Create project structure and virtual environment
2. Implement `config.py` — load and validate `config.json` (new format with site/tag/track_from objects, storage config)
3. Implement adapter base class (`adapters/base.py`) — abstract `SiteAdapter` interface
4. Implement dual state backend (`state.py`, `state_json.py`, `state_sqlite.py`) — load/save seen works via JSON or SQLite based on config
5. Create `config.example.json` (new format)
6. Set up logging with `RotatingFileHandler`

### Phase 2: AO3 Adapter
7. Implement `adapters/ao3.py` — login, search by relationship, extract work metadata, tag validation
8. Handle rate limiting (429 detection, delays between requests)
9. Wire into `main.py` with the async polling loop and `track_from` date filtering

### Phase 3: Email
10. Implement `email_builder.py` — HTML digest template with inline CSS (grouped by site and tag)
11. Implement `email_sender.py` — configurable SMTP with TLS/SSL support
12. Wire email send into `main.py` after successful search

### Phase 4: Tier 1 Site Adapters
13. Implement `adapters/fimfiction.py` — OAuth2 auth, official REST API, character tag search
14. Implement `adapters/xenforo.py` — RSS feed parsing for SpaceBattles and Sufficient Velocity
15. Verify all Tier 1 adapters work end-to-end with digest email

### Phase 5: Tier 2 Site Adapters (Experimental)
16. Implement `adapters/wattpad.py` — undocumented internal API, freeform tag search
17. Implement `adapters/quotev.py` — web scraping, freeform tag search
18. Implement `adapters/ffnet.py` — curl_cffi Cloudflare bypass, character pair filter
19. Add graceful failure handling for experimental adapters (skip on failure, log warning, continue)

### Phase 6: Scheduling & Polish
20. Create `run.bat` and `setup_scheduler.bat`
21. Add `.gitignore` (config.json, seen_works.json, scanner.log, __pycache__, venv)
22. Create README.md with setup instructions (covering all supported sites, auth requirements, track_from setup)
23. End-to-end testing across all adapters

---

## 14. Verification

1. **Config validation**: Run with a missing config, an invalid config (bad JSON, missing fields), a relationship missing `track_from`, and a valid config — verify correct error messages or successful load
2. **AO3 adapter**: Run with correct credentials — verify login succeeds and restricted works appear in results. Run with wrong credentials — verify clean error
3. **Fimfiction adapter**: Register OAuth2 app, search by character tags — verify results match the site's web UI
4. **XenForo adapter**: Monitor a SpaceBattles creative writing tag via RSS — verify new threads are detected
5. **Experimental adapters**: Test Wattpad/Quotev/FFNet — verify they return results when working and fail gracefully when blocked
6. **track_from filtering**: Add a tag with `track_from` set to yesterday. Verify only works posted after yesterday trigger notifications, even if older unseen works appear in results
7. **Search accuracy**: Manually check AO3 for a known relationship tag, compare script results against the web UI. Confirm restricted works are included
8. **State tracking (JSON mode)**: Run once to populate seen_works.json. Remove an ID from seen_works.json. Run again — verify the work is NOT re-flagged as new (because `date_posted` < `track_from`)
9. **State tracking (SQLite mode)**: Configure SQLite mode, point to a test database. Verify discoveries are written to the ScannerDiscoveries table
10. **Email delivery**: Send a test digest to your inbox. Verify the HTML renders correctly, links work, restricted indicator shows, works are grouped by site and tag
11. **SMTP flexibility**: Test with at least one SMTP provider to confirm the configurable approach works
12. **Task Scheduler**: Create the scheduled task via `setup_scheduler.bat`. Verify it appears in Task Scheduler. Wait for one triggered run and check `scanner.log` for a successful execution
13. **Error resilience**: Kill the SMTP connection mid-send — verify state was not updated and the next run re-finds the same works
14. **Adapter failure isolation**: Disable network access for one site — verify other sites' scans complete normally
