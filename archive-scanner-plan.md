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
| HTTP | aiohttp (ao3.py dependency) | Async HTTP client used internally by ao3.py |
| Email | smtplib + email.mime (stdlib) | No external dependency needed, works with any SMTP provider |
| Email Templates | string.Template (stdlib) | Lightweight HTML templating without pulling in Jinja2 |
| State | JSON file | Simple, no database needed — just a dict of ship → list of seen work IDs |
| Logging | logging (stdlib) | RotatingFileHandler for log rotation |
| Scheduling | Windows Task Scheduler | Native to Windows, no daemon process needed |

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
    "Character A/Character B",
    "Character C & Character D"
  ],
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
| `relationships` | List of AO3 relationship tags to monitor (exact tag names as they appear on AO3) |
| `search.max_pages` | How many pages of results to check per relationship (each page = 20 works) |
| `search.delay_between_requests_seconds` | Delay between AO3 requests to avoid rate limiting |

---

## 5. State File — `seen_works.json`

```json
{
  "Character A/Character B": [12345678, 12345600, 12345590],
  "Character C & Character D": [12345670, 12345655]
}
```

Each relationship maps to a list of work IDs that have already been seen and notified on. On each run, new work IDs are appended. The file is capped at 500 IDs per relationship to prevent unbounded growth — oldest IDs are trimmed when the cap is exceeded.

---

## 6. Core Logic — Polling Flow

```
1. LOAD config.json
   │
2. CREATE ao3.py Client, LOGIN with credentials
   │
3. LOAD seen_works.json (create empty if missing)
   │
4. FOR EACH relationship in config.relationships:
   │
   ├── BUILD WorkSearchOptions:
   │     relationship_names = [relationship]
   │     sort_column = "revised_at"
   │     sort_direction = "desc"
   │
   ├── SEARCH via client.work_search_pages()
   │     (up to max_pages pages)
   │     (delay between pages per config)
   │
   ├── FOR EACH work in results:
   │     ├── IF work.id NOT IN seen_works[relationship]:
   │     │     COLLECT: id, title, authors, summary,
   │     │             url, relationships, date_published,
   │     │             nwords, nchapters, is_complete,
   │     │             rating, is_restricted
   │     │     ADD to new_works list
   │     └── ELSE: SKIP (already seen)
   │
   ├── DELAY between relationships
   │
5. IF new_works is not empty:
   │  ├── BUILD HTML digest email
   │  │     grouped by relationship
   │  │     with work metadata and direct links
   │  ├── SEND via SMTP
   │  └── LOG: "Sent digest with N new works"
   │
6. UPDATE seen_works.json with all new IDs
   │  (trim to 500 per relationship)
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
│   ├── ao3_client.py         # AO3 login + search logic using ao3.py
│   ├── email_builder.py      # Builds HTML digest email from new works
│   ├── email_sender.py       # SMTP send logic
│   ├── state.py              # Load/save seen_works.json
│   └── config.py             # Load and validate config.json
├── config.json               # User configuration (gitignored)
├── config.example.json       # Example config checked into repo
├── seen_works.json           # State file (gitignored)
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
    seen = load_seen_works()
    
    client = ao3.Client()
    await client.login(config.ao3.username, config.ao3.password)
    
    all_new_works = {}
    
    for ship in config.relationships:
        options = WorkSearchOptions(
            relationship_names=[ship],
            sort_column="revised_at",
            sort_direction="desc"
        )
        
        new_works = []
        async for page in client.work_search_pages(options, start=1, stop=config.search.max_pages + 1):
            for work in page.results:
                if work.id not in seen.get(ship, []):
                    new_works.append(extract_work_info(work))
            await asyncio.sleep(config.search.delay_between_requests_seconds)
        
        if new_works:
            all_new_works[ship] = new_works
    
    if all_new_works:
        html = build_digest_email(all_new_works)
        send_email(config, html, total_count(all_new_works))
    
    save_seen_works(seen, all_new_works)

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
```

ao3.py pulls in aiohttp as a transitive dependency. smtplib, email, json, logging, and asyncio are all stdlib — no additional packages needed.

---

## 11. Error Handling

| Scenario | Behaviour |
|----------|-----------|
| AO3 login failure | Log error, exit with code 1. No email sent |
| AO3 rate limited (HTTP 429) | Log warning, skip remaining searches for this run, send digest of whatever was found so far |
| AO3 request timeout | Log warning, skip that relationship, continue with others |
| SMTP connection failure | Log error, exit with code 1. seen_works.json is NOT updated so the same works will be retried next run |
| config.json missing | Log error, exit with code 1 |
| config.json invalid | Log error with details, exit with code 1 |
| seen_works.json missing | Create empty state, treat all found works as new |
| seen_works.json corrupted | Log warning, reset to empty state |
| No new works found | Log info, exit cleanly. No email sent |

The critical design choice: **seen_works.json is only updated after a successful email send**. This ensures that if the email fails, the same works will be picked up on the next run.

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
2. Implement `config.py` — load and validate `config.json`
3. Implement `state.py` — load/save `seen_works.json` with ID cap
4. Create `config.example.json`
5. Set up logging with `RotatingFileHandler`

### Phase 2: AO3 Integration
6. Implement `ao3_client.py` — login, search by relationship, extract work metadata
7. Handle rate limiting (429 detection, delays between requests)
8. Wire into `main.py` with the async polling loop

### Phase 3: Email
9. Implement `email_builder.py` — HTML digest template with inline CSS
10. Implement `email_sender.py` — configurable SMTP with TLS/SSL support
11. Wire email send into `main.py` after successful search

### Phase 4: Scheduling & Polish
12. Create `run.bat` and `setup_scheduler.bat`
13. Add `.gitignore` (config.json, seen_works.json, scanner.log, __pycache__, venv)
14. Create README.md with setup instructions
15. End-to-end testing

---

## 14. Verification

1. **Config validation**: Run with a missing config, an invalid config (bad JSON, missing fields), and a valid config — verify correct error messages or successful load
2. **AO3 login**: Run with correct credentials — verify login succeeds and restricted works appear in results. Run with wrong credentials — verify clean error
3. **Search accuracy**: Manually check AO3 for a known relationship tag, compare script results against the web UI. Confirm restricted works are included
4. **State tracking**: Run once to populate seen_works.json. Add a fic manually to AO3 search results (or remove an ID from seen_works.json). Run again — verify only the "new" work triggers a notification
5. **Email delivery**: Send a test digest to your inbox. Verify the HTML renders correctly in your email client, links work, restricted indicator shows
6. **SMTP flexibility**: Test with at least one SMTP provider to confirm the configurable approach works
7. **Task Scheduler**: Create the scheduled task via `setup_scheduler.bat`. Verify it appears in Task Scheduler. Wait for one triggered run and check `scanner.log` for a successful execution
8. **Error resilience**: Kill the SMTP connection mid-send — verify seen_works.json was not updated and the next run re-finds the same works
