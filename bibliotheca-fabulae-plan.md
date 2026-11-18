# Bibliotheca Fabulae — Complete Project Plan

## Overview

A self-hosted personal fanfiction archive and reader. Downloads fics from bookmarks/favourites across 100+ supported sites (AO3, FFNet, Wattpad, SpaceBattles, Royal Road, etc.), stores them locally with full metadata, and provides a private web interface for reading, searching, and organising your collection. Integrates with the separate Archive Scanner project for new work discovery by relationship tag. Includes a multi-phase text-to-speech reader with gender-differentiated and character-specific voice cloning.

**Core principles:**
- Personal archive — single user, private, never shared publicly
- Multi-source — any supported site, configurable per-source
- Preservation-first — deleted/privated fics are kept, version history is retained
- Full-featured reader — responsive, TTS with character voices, dark mode
- Cross-platform — runs on Windows (IIS) or Linux (Docker)

---

## Tech Stack

- **Frontend:** React (mobile-first responsive SPA, Tailwind CSS)
- **Backend:** Python / FastAPI
- **Database:** SQLite (WAL mode, FTS5 for full-text search)
- **ORM:** SQLAlchemy with Alembic for migrations
- **Download engine:** FanFicFare (100+ site support, EPUB/HTML output)
- **AO3 client:** ao3.py (authenticated access, search, bookmark sync)
- **Dialogue NLP:** BookNLP (dialogue attribution, character identification, gender inference)
- **TTS — server-side:** edge-tts (Microsoft Edge neural voices, 300+ voices, free)
- **TTS — browser fallback:** Web Speech API
- **TTS — voice cloning:** Coqui XTTS v2 (open-source, zero-shot cloning from 6-15s reference clip)
- **Audio separation:** Demucs (isolates speech from music/SFX in voice clips)
- **Scheduler:** APScheduler (in-process, cross-platform)
- **Email:** smtplib (stdlib, configurable SMTP)
- **Print PDF:** WeasyPrint (HTML to print-ready PDF with book formatting)
- **API docs:** Auto-generated OpenAPI (built into FastAPI)
- **Deployment:** IIS reverse proxy (Windows) or Docker Compose + Nginx (Linux)
- **SSL:** Cloudflare Zero Trust (perimeter security) + WinACME (certificate)
- **Testing:** pytest (backend only)

**Why this stack:** FanFicFare and ao3.py are both Python, making FastAPI the natural backend choice. FanFicFare handles the complexity of scraping 100+ sites with different layouts, anti-bot measures, and authentication flows — wrapping it avoids reinventing that wheel. SQLite with FTS5 handles full-text search across thousands of fics without an external search engine. edge-tts provides high-quality neural voices for free without API keys. Coqui XTTS v2 runs locally for voice cloning without cloud dependencies.

**SQLite notes:** With a single user and background tasks serialised via APScheduler, write contention is minimal. WAL mode allows concurrent reads during background sync/download operations. FTS5 virtual tables provide fast full-text search with ranking. Version history is stored as compressed chapter snapshots — rebuild on demand rather than duplicating files.

---

## Repository Structure

Monorepo — single repository, everything versioned together.

```
BibliothecaFabulae/
  ├── frontend/              (React app)
  ├── backend/               (FastAPI app)
  │   ├── api/               (route handlers)
  │   ├── core/              (config, auth, scheduler)
  │   ├── downloader/        (FanFicFare wrapper, site adapters)
  │   ├── scanner/           (Archive Scanner integration)
  │   ├── tts/               (TTS engine, dialogue parsing, voice management)
  │   ├── models/            (SQLAlchemy models)
  │   └── services/          (business logic)
  ├── alembic/               (database migrations)
  ├── iis/                   (IIS configuration files)
  ├── nginx/                 (Nginx config for Docker deployment)
  ├── docker-compose.yml
  ├── deploy.sh              (Linux deployment script)
  ├── deploy.bat             (Windows deployment script)
  └── README.md
```

---

## Server Hardware

The application is designed for a TRIGKEY Mini PC with a Ryzen 5 5560U (6 core / 12 thread, no discrete GPU).

- **Phases 1-3 (TTS)** — run comfortably on this hardware. edge-tts is an API call, BookNLP processes chapters in seconds on CPU.
- **Phase 4 (voice cloning)** — Coqui XTTS runs on CPU but is slow (~2-5 minutes per chapter). Heavy voice cloning work is offloaded to a separate GPU-equipped machine configured in the UI.
- **Storage** — fic files and database stored on a dedicated 50GB partition, path configurable in settings.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│          Cloudflare Zero Trust (perimeter security)          │
├─────────────────────────────────────────────────────────────┤
│    IIS (Windows) or Nginx (Linux) — reverse proxy + SSL     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌────────────────────────────────┐     │
│  │  React SPA   │    │       FastAPI Backend           │     │
│  │  (Frontend)  │◄──►│                                │     │
│  │              │    │  ├─ Auth (JWT + TOTP 2FA)      │     │
│  │  - Archive   │    │  ├─ Archive API                │     │
│  │  - Reader    │    │  ├─ Downloader Service         │     │
│  │  - Scanner   │    │  │   (FanFicFare wrapper)      │     │
│  │  - TTS       │    │  ├─ Scanner Integration        │     │
│  │  - Sources   │    │  │   (ao3.py + multi-site)     │     │
│  │  - Settings  │    │  ├─ TTS Engine                 │     │
│  └──────────────┘    │  │   (edge-tts + BookNLP)      │     │
│                      │  ├─ Voice Cloning Service      │     │
│                      │  │   (Coqui XTTS v2)           │     │
│                      │  ├─ Notification Service       │     │
│                      │  │   (UI + SMTP email)         │     │
│                      │  ├─ Scheduler (APScheduler)    │     │
│                      │  │   ├─ Bookmark sync          │     │
│                      │  │   ├─ Fic update checks      │     │
│                      │  │   └─ Scanner polling        │     │
│                      │  └─ Print PDF Generator        │     │
│                      │      (WeasyPrint)              │     │
│                      └──────────────┬─────────────────┘     │
│                                     │                       │
│                      ┌──────────────▼─────────────────┐     │
│                      │   SQLite (WAL + FTS5)          │     │
│                      │                                │     │
│                      │  + File Storage (configurable) │     │
│                      │    ├─ HTML (web reader)        │     │
│                      │    ├─ EPUB (offline/e-reader)  │     │
│                      │    └─ Audio cache (MP3)        │     │
│                      └────────────────────────────────┘     │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Remote GPU Machine (optional, configurable)       │     │
│  │  Coqui XTTS v2 worker for character voice cloning  │     │
│  │  Queues jobs when unavailable, processes on connect │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Archive Scanner (separate project)                │     │
│  │  Shared SQLite DB when storage_mode = "sqlite"     │     │
│  │  Independent JSON mode when standalone              │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Navigation Structure

Collapsible sidebar on desktop, hamburger menu on mobile.

```
📚 Archive
   ├── All Fics
   ├── Favourites
   └── Collections
🔍 Discover (Scanner)
   ├── New Works Feed
   └── Tracked Tags
📥 Import
   ├── Add by URL
   └── Bookmark Sync
🔔 Notifications (badge count)
📊 Storage
⚙️ Settings
   ├── Sources & Auth
   ├── Scanner Config
   ├── TTS & Voices
   ├── GPU Machine
   ├── Notifications
   ├── Storage Path
   ├── Backup
   └── Account
```

---

## Data Model

Fics are the central entity. Each fic has metadata from its source site, local organisational data (tags, favourite status, collections), file storage paths, and version history. Audio is generated on demand and cached according to storage budget rules.

```
Fics
  ├── source_site (ao3, ffnet, wattpad, etc.)
  ├── source_id (site-specific work ID)
  ├── source_url
  ├── title, summary, language
  ├── rating, warnings, categories
  ├── word_count, chapter_count, is_complete
  ├── date_published, date_updated (at source)
  ├── date_archived, date_last_synced (local)
  ├── is_favourite (boolean, local-only)
  ├── is_source_available (boolean — false if deleted/privated)
  ├── reading_position (chapter_index, paragraph_index, char_offset)
  ├── reading_status (none | reading — auto-managed)
  ├── html_path, epub_path
  ├── import_method (bookmark_sync | manual_url | scanner)
  └── Chapters
        ├── chapter_number, title
        ├── html_content (current version in file)
        └── ChapterVersions
              ├── content (compressed snapshot)
              ├── version_date
              └── word_count

Authors
  ├── name, source_site, source_id
  ├── profile_url
  └── FicAuthors (many-to-many with Fics)

Fandoms
  ├── name, source_site
  └── FicFandoms (many-to-many with Fics)

Tags
  ├── name, tag_type (source | user_defined)
  └── FicTags (many-to-many with Fics)

Relationships (ships)
  ├── name, relationship_type (romantic | platonic)
  ├── source_site
  └── FicRelationships (many-to-many with Fics)

Characters
  ├── name, source_site
  ├── gender (for TTS voice assignment)
  └── FicCharacters (many-to-many with Fics)
        └── is_pov (optional, for TTS narrator assignment)

Collections (user-defined groupings)
  ├── name, description, sort_order
  └── FicCollections (many-to-many with Fics)

Sources (enabled site configurations)
  ├── site_key (ao3, ffnet, wattpad, etc.)
  ├── is_enabled
  ├── auth_type (credentials | cookies | session | none)
  ├── auth_data (encrypted)
  ├── auth_status (valid | expired | unknown)
  ├── auth_last_checked
  ├── sync_interval_bookmarks (cron expression, default daily)
  ├── sync_interval_updates (cron expression, default daily)
  └── rate_limit_config (delay between requests, max pages)

ScannerTags (tracked relationship tags for discovery)
  ├── source_site
  ├── tag_name (validated against site's tag system)
  ├── track_from (user-set date — only notify on works posted after this)
  ├── scan_interval (cron expression, default hourly)
  ├── last_scanned
  └── SeenWorks
        ├── source_id
        ├── date_first_seen
        └── date_posted (to distinguish new vs updated)

ScannerDiscoveries (new works found by scanner)
  ├── scanner_tag_id
  ├── source_site, source_id, source_url
  ├── title, authors, summary
  ├── word_count, chapter_count, is_complete
  ├── rating, is_restricted
  ├── date_posted
  ├── is_archived (true if user one-clicked to archive)
  └── is_dismissed (true if user dismissed from feed)

Notifications
  ├── type (fic_updated | sync_complete | sync_failed |
  │         auth_required | storage_low | fic_unavailable |
  │         scanner_new_works | audio_ready)
  ├── title, message, severity
  ├── is_read
  ├── related_fic_id (nullable)
  ├── created_at
  └── email_sent (boolean)

NotificationPreferences
  ├── notification_type
  ├── ui_enabled (boolean)
  └── email_enabled (boolean)

VoiceProfiles (character voice configurations for TTS Phase 4)
  ├── character_name
  ├── source (manual_upload | clip_search | game_extraction)
  ├── reference_clip_path
  ├── clip_duration
  ├── is_validated (user approved after preview)
  └── created_at

AudioCache
  ├── fic_id, chapter_id
  ├── audio_type (basic | gendered | character)
  ├── file_path, file_size
  ├── is_favourite_fic (determines retention)
  ├── last_listened
  └── created_at

AudioStorageConfig
  ├── max_budget_mb
  ├── favourite_retention (permanent)
  └── cache_ttl_days (default 7)

GPUMachineConfig
  ├── hostname, port
  ├── is_enabled
  ├── last_seen_online
  └── GPUJobQueue
        ├── fic_id, chapter_id
        ├── status (queued | processing | complete | failed)
        ├── requested_at, completed_at
        └── voice_profiles_needed (JSON list)

BookmarkSyncState
  ├── source_site
  ├── last_synced
  ├── last_bookmark_id (for pagination/resume)
  └── total_bookmarks, total_synced

Settings
  ├── storage_path (configurable partition path)
  ├── theme (dark | light, default dark)
  ├── smtp_host, smtp_port, smtp_username, smtp_password (encrypted)
  ├── smtp_use_tls, smtp_from_address, smtp_to_address
  ├── backup_location, backup_schedule
  └── setup_complete (boolean)
```

**Bookmark metadata import:** When syncing AO3 bookmarks, the system imports bookmark-specific metadata (private/public status, rec status, bookmark tags, bookmark notes). The rec status is stored as source metadata, queryable via bulk operations for mapping to local favourites. FFNet favourites and other site-specific bookmark metadata are similarly preserved.

**Version history:** Only the latest chapter content is stored as files (HTML + EPUB). When a chapter update is detected, the previous content is compressed and stored as a ChapterVersion row in SQLite. To read an older version, the system reconstructs it from the database on demand — this adds a few seconds of processing time but keeps file storage lean.

**Audio storage:** Audio files (MP3) are cached according to the AudioStorageConfig budget. Fics marked as favourites retain their audio permanently. All other audio is purged when the cache TTL expires or the storage budget is exceeded (oldest/least-listened first). Audio is regenerated on demand.

---

## Source Configuration

All FanFicFare-supported sites (100+) are available as sources. Each site appears as a toggle in the Sources settings page. When enabled, the UI shows auth fields appropriate to that site.

### Authentication by site type

| Auth Type | Sites | Setup |
|-----------|-------|-------|
| Credentials (username/password) | AO3, FicWad, most archives | Enter in settings, stored encrypted |
| Browser cookies / session | FFNet (Cloudflare), Wattpad, Scribble Hub | Paste cookies from browser, guided instructions in UI |
| API key | Sites with APIs | Enter key in settings |
| None | Sites with public access | No setup needed |

### Auth health monitoring

On each sync/download, the system validates that credentials are still working. If authentication fails:
1. The source's `auth_status` is set to `expired`
2. A notification is raised (UI + email if configured)
3. The notification includes instructions for how to re-authenticate for that specific site
4. Downloads from that source are paused until re-authenticated

---

## Scanner Integration

The Archive Scanner is a separate standalone project. Bibliotheca Fabulae integrates with it via a shared SQLite database.

### How it works

1. The Archive Scanner's `config.json` has a `storage` section:
   ```json
   {
     "storage": {
       "mode": "json",
       "sqlite_path": ""
     }
   }
   ```
2. When `mode` is `"json"` (default), the Scanner operates standalone with `seen_works.json`
3. When `mode` is `"sqlite"`, the Scanner reads/writes to the Bibliotheca Fabulae database at the configured path
4. The Bibliotheca Fabulae settings page shows the database path and can generate the Scanner config snippet

### Multi-site scanning

The Scanner supports multiple sites via a site adapter architecture. Not all sites support pairing/relationship searches — the scanner adapts to each site's capabilities.

#### Supported scanner sites

**Tier 1 — Stable, ship with MVP:**

| Site | Search Type | Method | Tag Format |
|------|-------------|--------|------------|
| **AO3** | Relationship tags | ao3.py (authenticated) | Canonical tags: `Character A/Character B` |
| **Fimfiction** | Character tags | Official REST API (OAuth2) | Character IDs: `#fluttershy #rainbow-dash` |
| **SpaceBattles** | Fandom/character tags | XenForo RSS feeds | Forum tags: `worm`, `taylor-hebert` |
| **Sufficient Velocity** | Fandom/character tags | XenForo RSS feeds | Forum tags (same as SpaceBattles) |

**Tier 2 — Experimental, may break:**

| Site | Search Type | Method | Tag Format | Caveat |
|------|-------------|--------|------------|--------|
| **Wattpad** | Freeform ship-name tags | Undocumented internal API | Concatenated: `drarry`, `bakudeku` | No official API, can break without notice |
| **Quotev** | Freeform ship-name tags | Web scraping | Freeform: `drarry`, `draco-x-harry` | Freeform tags, inconsistent naming |
| **FFNet** | Character pair filter | Browser cache / curl_cffi | Numeric character IDs + pairing checkbox | Cloudflare arms race, high maintenance |

**Not supported for scanning** (still supported as download sources via FanFicFare):
Royal Road, Scribble Hub, Adult-FanFiction.org, FicWad — these sites lack the tag/pairing metadata needed for relationship scanning.

#### Site-specific notes

- **SpaceBattles / Sufficient Velocity**: Share the same XenForo codebase — one adapter covers both. Can filter by fandom and character but **not by ship/pairing** (forums don't have relationship tags). Useful for "new fic in [fandom]" monitoring.
- **Wattpad / Quotev**: Use freeform tags — the same ship might be tagged multiple ways (`drarry`, `draco-x-harry`, `harryxdraco`). The UI allows adding multiple variant tags per ship to cover all naming conventions.
- **FFNet**: Has actual character pair filters but Cloudflare anti-bot protection makes access fragile. Marked as experimental with a "may require periodic maintenance" warning in the UI.
- **Fimfiction**: Official API with OAuth2 auth, structured character tags, date sorting. Best integration after AO3. MLP-only.

#### Adding a tracked tag

When adding a tracked tag in the Bibliotheca Fabulae UI:
1. Select the source site — the UI shows which sites support scanning and their tier (stable/experimental)
2. The UI adapts its input to that site's tag system:
   - AO3: autocomplete against canonical relationship tags
   - Fimfiction: character tag picker from API
   - SpaceBattles/SV: fandom and character tag selection from forum tag list
   - Wattpad/Quotev: freeform text input with option to add multiple variant tags
   - FFNet: fandom dropdown → character pair selection from fandom character list
3. The tag is validated before saving to ensure it will return results
4. **Set "Track from" date** — the user specifies the starting point for notifications. Options:
   - **"From now"** (default) — only notify on works posted after this moment
   - **Custom date** — a date picker to set any past date (e.g., "notify me about anything posted after 1st January 2026")
   - This is a required step — the scanner never guesses or runs a silent baseline scan

### New vs updated fic detection

The Scanner checks `date_posted` against the tag's `track_from` date in addition to work ID. A work is only flagged as "new" if:
- Its ID has never been seen before, **AND**
- Its `date_posted` is after the tag's `track_from` date

This prevents the ID cap trim from causing false "new" notifications for updated fics, and eliminates the cold-start problem entirely — the user defines the boundary, not the system.

---

## TTS Reader

Text-to-speech is built in four phases during development. All four phases are included in the MVP release. Audio generation is on-demand and user-initiated.

### Audio generation flow

1. User clicks the audio button on a fic or chapter
2. A modal presents the available audio types:
   - **Basic** — single narrator voice (edge-tts)
   - **Gendered** — male/female/narrator voices (edge-tts + BookNLP dialogue attribution)
   - **Character voices** — cloned voices per character (Coqui XTTS v2, requires GPU machine)
3. For Basic/Gendered: a loading bar shows progress, audio is ready within seconds to a minute
4. For Character voices:
   - System checks if the GPU machine is available
   - If **available**: shows which character voices are needed, flags characters without voice profiles, starts processing
   - If **unavailable**: informs the user, queues the job, notifies when the GPU machine comes online and audio is complete
5. User is notified when audio generation is complete (if it takes more than a few seconds)
6. Audio plays with play/pause/chapter skip controls
7. Reading position is saved whether reading text or listening to audio

### Dialogue attribution pipeline

1. **BookNLP** processes each chapter at import time to identify:
   - Quoted dialogue and the speaking character
   - Character names and genders
   - Narration vs dialogue segments
2. Output is an ordered list of `{text, speaker, gender, start_offset, end_offset}` chunks
3. Each chunk is assigned a voice:
   - Narration → neutral narrator voice
   - Male character dialogue → male voice (e.g., `en-US-GuyNeural`)
   - Female character dialogue → female voice (e.g., `en-US-JennyNeural`)
   - Specific character → cloned voice profile (Phase 4)
4. Characters without voice profiles fall back to gendered voices
5. User can manually correct dialogue attribution if BookNLP gets it wrong

### Voice profile management

Three methods for creating character voice profiles:

**Manual upload:**
- Upload a 6-15 second clean audio clip of the character's voice
- Preview the cloned voice before saving
- Best for characters you have clean audio for already

**Clip search + Demucs isolation (Option A):**
- Search for a character by name, system finds candidate voice clips online
- Demucs AI audio separation strips background music and SFX
- Preview candidate clips, approve the best one
- System clones from the approved clip

**Game voice file extraction (Option B):**
- Point the app at a game's audio directory (e.g., Marvel's Avengers, Midnight Suns)
- System scans for isolated character voice files
- Preview and approve clips for each character
- Cleanest source — game dialogue is already isolated

### GPU machine configuration

Character voice cloning (Phase 4) requires significant processing power. This work is offloaded to a separate local machine with a GPU.

- Configured in Settings → GPU Machine (hostname/IP, port)
- System checks availability before starting character voice jobs
- If unavailable: job is queued, user is informed, notification sent when machine comes online and job completes
- The GPU machine runs a lightweight worker service that accepts cloning requests from the main server

---

## Reading Position & Status

Reading position is automatically saved for both text reading and audio listening.

**Position tracking:**
- Saved as `{chapter_index, paragraph_index, char_offset}`
- Updated as you scroll (text) or as audio plays
- Resuming opens at the exact saved position

**Status tracking (minimal, mostly automatic):**
- **No status** (default) — imported fics, assumed already read on the source platform
- **Reading** — automatically set when a reading position is saved, cleared when you reach the end of the last chapter

**Completion badge (from source metadata):**
- Complete
- In Progress
- Hiatus (if detectable from source)

---

## Favourites & Tags

### Favourites

Favourites are a first-class concept, separate from tags:
- Dedicated heart/star toggle on each fic
- Local-only — does not sync back to source sites
- Own filtered view in the archive
- Affects audio retention (favourite fics keep audio permanently)

### Custom tags

User-defined tags for organisation:
- Create any tag (e.g., "NSFW", "Comfort Fic", "Re-read Later")
- Apply to individual fics or in bulk
- Filter/search by tags throughout the archive
- Separate from source tags (AO3 tags, FFNet categories, etc.) which are preserved as-is

### Bulk operations

A dedicated bulk operations page:
1. **Filter** — narrow down fics using any combination of metadata, source tags, user tags, source site, favourite status, etc.
2. **Select** — select all filtered results or pick individually
3. **Action** — choose what to apply:
   - Mark/unmark as favourite
   - Add tag / remove tag
   - Add to collection / remove from collection

**Example use case:** Filter by "AO3 bookmarks where rec = true" → select all → "Mark as Favourite" to migrate AO3 recs to local favourites.

---

## Search & Filtering

### Full-text search
- SQLite FTS5 indexes all chapter content
- Search within fic content to find that one scene you remember
- Results ranked by relevance with highlighted snippets

### Metadata filters
- Fandom
- Relationship / ship
- Characters
- Rating (General, Teen, Mature, Explicit, etc.)
- Warnings
- Completion status (Complete, In Progress, Hiatus)
- Word count range
- Chapter count range
- Author
- Source site
- Language
- Source tags (preserved from origin site)
- User tags (locally created)
- Collections
- Favourite status
- Source availability (available / deleted / privated)

### Sort options
- Date added to archive
- Date updated (at source)
- Word count
- Title (alphabetical)
- Author (alphabetical)

### Pagination
- All list views are paginated with lazy loading
- Designed for thousands of fics without performance issues

---

## Export

### EPUB download
- Available on every fic — the EPUB is already stored as part of the archive
- Single fic or bulk download (zip of EPUBs)

### Print-ready PDF
- "Print Book" export option on each fic
- Configurable page size (A5, 6x9", A4, Letter, etc.)
- Proper book formatting:
  - Gutter margins for binding side
  - Page numbers
  - Running headers (fic title / author)
  - Chapter title pages
  - Title page with fic metadata
  - Widow/orphan control
- Generated via WeasyPrint (HTML to PDF)
- Suitable for home printing (duplex A5 booklets) or print-on-demand services (Lulu, BookVault)

---

## Notifications

All notifications are available via two channels: in-app UI and email (SMTP). Each notification type is independently configurable per channel.

### Notification types

| Type | Description |
|------|-------------|
| Fic updated | A fic in the archive has new chapters |
| Sync complete | A bookmark sync finished successfully |
| Sync failed | A bookmark sync encountered errors |
| Auth required | A source site needs re-authentication |
| Storage low | Storage usage is approaching the partition limit |
| Fic unavailable | A fic has been deleted/privated at source |
| Scanner — new works | New fics discovered for a tracked tag |
| Audio ready | TTS audio generation is complete |

### Email specifics
- Configurable SMTP (works with Gmail, Outlook, SendGrid, self-hosted, etc.)
- Scanner email digests: short digests are inline, longer ones attach an HTML summary
- Each notification type can be enabled/disabled for email independently

### UI specifics
- Badge count in navigation for unread notifications
- Notification feed page with mark-as-read and dismiss
- Deleted/privated fics get a visual badge on the fic card in the archive

---

## Storage Management

### Storage dashboard
- Total used / available on the configured partition
- Breakdown by category:
  - HTML fic files
  - EPUB files
  - SQLite database (including version history)
  - Audio cache
- Low storage warning threshold (configurable, notification raised when exceeded)

### Audio storage budget
- Configurable maximum audio storage (e.g., 10GB)
- Favourite fics retain audio permanently
- Non-favourite audio purged when cache TTL expires (default 7 days) or budget is exceeded
- Oldest/least-listened audio purged first
- Purged audio regenerated on demand

### File storage path
- Configurable in settings — point to any directory/partition
- Default validation on startup that the path exists and is writable

---

## Security

- **Cloudflare Zero Trust** — perimeter security, device/identity checks before traffic reaches the server
- **Single-user login with TOTP 2FA** — app-level authentication behind Cloudflare
- **JWT authentication** with short-lived access tokens + refresh tokens
- **Session management** — configurable "remember device" period to reduce 2FA frequency
- **bcrypt** password hashing (minimum 12 rounds)
- **Credential encryption** — all source site credentials and SMTP passwords encrypted at rest (AES-256)
- **Input sanitisation** on all endpoints (Pydantic models)
- **CORS** locked to known origins
- **Rate limiting** on all endpoints
- **SQL injection prevention** via ORM (SQLAlchemy)
- **Rotating log files** — prevent disk fill from logging

---

## Voice Cloning & Content Usage

The app ships with **no pre-loaded voice profiles**. Users create their own for personal use.

**Personal use is fine:**
- Clone any character voice you want for your own private use (JARVIS, Cortana, Emma Frost, etc.)
- Use reference clips from games, cartoons, movies, audiobooks
- This is a personal, self-hosted tool — the same as hobbyist projects widely shared online

**Don't distribute or sell:**
- Do not ship or distribute the app with pre-loaded voice models based on copyrighted characters
- Do not distribute trained voice models or generated audio of copyrighted character voices
- Do not sell access to cloned character voices
- If the app is ever open-sourced, it ships clean — users bring their own voice data

**Fanfiction archival:**
- AO3's Terms of Service permit personal archival downloads
- This is a private, single-user tool — no redistribution of fic content
- Respect rate limits and site-specific terms when downloading

This same guidance applies to the Virtual Assistant project's voice features.

---

## Deployment

### Windows (IIS)

The application runs behind IIS as a reverse proxy, allowing domain binding with Cloudflare Zero Trust.

```
IIS Site
  └── Reverse proxy → FastAPI (uvicorn on localhost:8000)
      └── React SPA served as static files by FastAPI
```

Setup:
1. Install Python, clone repo, install dependencies
2. Configure IIS URL Rewrite for reverse proxy to localhost:8000
3. Run FastAPI via a Windows Service (NSSM or similar)
4. Configure Cloudflare Zero Trust for the domain

### Linux (Docker Compose)

```bash
git clone <repo>
docker compose up -d
```

```
Docker Compose
  ├── nginx (reverse proxy + SSL)
  ├── backend (FastAPI + React static files)
  └── volumes
       └── data/ (SQLite DB + fic files, mapped to host)
```

### Updates

**Windows (`deploy.bat`):**
```bat
@echo off
cd /d "%~dp0"
git pull
pip install -r requirements.txt
python -m pytest
echo Restarting service...
net stop BibliothecaFabulae
net start BibliothecaFabulae
```

**Linux (`deploy.sh`):**
```bash
#!/bin/bash
git pull
docker compose run --rm backend pytest
docker compose build
docker compose up -d
```

Tests run before restart/rebuild — deployment aborts if tests fail.

---

## Implementation Phases

### Phase 1 — Foundation
- Project scaffolding (React + FastAPI + SQLite)
- Docker Compose setup + IIS configuration files
- Database migrations framework (Alembic)
- SQLite configured in WAL mode + FTS5
- **First-run setup wizard:**
  1. Create account (username, password, TOTP 2FA setup)
  2. Configure storage path (point to partition)
  3. Configure SMTP (optional, skip for UI-only notifications)
  4. Summary and go to archive
- Single-user JWT authentication with TOTP 2FA
- Session management with configurable device remember period
- Responsive layout with collapsible sidebar (hamburger on mobile)
- Tailwind CSS with dark/light theme toggle (dark default)
- Notification system (UI + email framework)
- Notification preferences page (per-type, per-channel toggles)
- Storage dashboard (usage breakdown, low storage warnings)
- Settings pages (storage path, theme, account, backup)
- Health endpoint (`/health`)
- Rotating log files

### Phase 2 — Source Management & Downloads
- Source configuration page (toggle sites, enter auth per site)
- Auth health monitoring (validate credentials on each sync, notify on expiry)
- FanFicFare integration (wrap CLI/library for downloads)
- **Manual URL import:**
  - Paste a fic URL into the UI
  - System identifies the source site, downloads the fic
  - Extracts metadata, stores HTML + EPUB
  - Indexes content in FTS5
- **Bookmark sync engine:**
  - AO3: ao3.py authenticated bookmark pull (preserves rec/private/tags/notes)
  - FFNet: FanFicFare favourites pull
  - Other sites: site-specific adapters where supported
  - Configurable sync interval per source (default daily)
  - Sync state tracking (resume from last position)
  - Notifications on complete/failure
- Fic update checker (detect new chapters, download updates)
- Version history (compress previous chapter content to SQLite on update)
- Deleted/privated fic detection (badge + configurable notification)
- Rate limiting per source (configurable delays, respects site limits)

### Phase 3 — Archive UI
- Archive browse page (paginated grid/list view of all fics)
- Fic detail page (metadata, chapters, source link, availability badge)
- **Web reader:**
  - Chapter-by-chapter reading view
  - Clean, distraction-free layout
  - Reading position auto-saved on scroll
  - "Reading" status auto-set when position exists
  - Resume reading from saved position
- Full-text search (FTS5 with ranked results and snippets)
- Metadata filtering (all filters listed in Search & Filtering section)
- Sort options
- Favourites (heart/star toggle, filtered view)
- Custom tags (create, apply, filter)
- Collections (create, add fics, filtered view)
- Bulk operations page (filter → select → action)
- EPUB download (single and bulk zip)

### Phase 4 — Scanner Integration
- Scanner integration settings (database path display, config snippet generator)
- Site adapter architecture:
  - Abstract base adapter interface for site-specific scanning
  - **Tier 1 adapters (stable):** AO3 (ao3.py), Fimfiction (official REST API), SpaceBattles (XenForo RSS), Sufficient Velocity (XenForo RSS)
  - **Tier 2 adapters (experimental):** Wattpad (undocumented API), Quotev (web scraping), FFNet (curl_cffi / browser cache)
  - Tier 2 sites shown with "experimental — may break" warning in UI
- Tracked tags management page:
  - Add tag: select source site (shows tier/stability) → site-aware autocomplete/validation → set "Track from" date → save
  - "Track from" options: "From now" (default) or custom date picker
  - Wattpad/Quotev: option to add multiple variant tags per ship (e.g., `drarry` + `draco-x-harry`)
  - FFNet: fandom dropdown → character pair selection
  - Edit scan interval per tag
  - Edit "Track from" date (with warning that changing to an earlier date may surface old works)
  - Remove tracked tags
- Scanner discovery feed (new works grouped by tag, with metadata, unified across all scanned sites)
- One-click archive from discovery feed
- Dismiss unwanted discoveries
- Scanner email digest (inline for short, attached HTML for long)

### Phase 5 — Print Export
- Print-ready PDF generation (WeasyPrint)
- Book formatting options UI:
  - Page size selection (A5, 6x9", A4, Letter)
  - Margin configuration (including gutter for binding)
- Generated PDF includes: title page, chapter title pages, running headers, page numbers
- Download generated PDF

### Phase 6 — TTS Phase 1 (Basic)
- Web Speech API integration (browser-native, zero dependencies)
- Play/pause/resume controls
- Chapter navigation during playback
- Reading position saved from audio playback
- Voice selection from browser's available voices

### Phase 7 — TTS Phase 2 (Better Quality)
- edge-tts server-side integration
- Audio generation on demand (user clicks audio button, selects "Basic")
- Loading bar during generation
- Pre-generated MP3 per paragraph, served via HTML5 Audio API
- Configurable narrator voice from edge-tts's 300+ voice library
- Audio cache with storage budget management
- Notification when generation complete (if slow)

### Phase 8 — TTS Phase 3 (Gendered Voices)
- BookNLP integration for dialogue attribution at import time
- Chapter parsing: identify narration, male dialogue, female dialogue
- Segment-based audio generation (different edge-tts voice per segment)
- Audio type selection modal (Basic vs Gendered)
- Manual correction UI for dialogue attribution errors
- Character gender metadata stored and editable

### Phase 9 — TTS Phase 4 (Character Voices)
- Coqui XTTS v2 integration
- GPU machine configuration page (hostname, port, availability check)
- Job queue for GPU work (queue when machine unavailable, process when online)
- Voice profile management page:
  - Manual upload (6-15s reference clip, preview before saving)
  - Clip search + Demucs isolation (search, isolate speech, preview, approve)
  - Game voice file extraction (point to game directory, scan, preview, approve)
- Character voice assignment (map characters to voice profiles)
- Audio type selection modal updated (Basic vs Gendered vs Character Voices)
- Shows GPU machine availability status and needed voice profiles before generating
- Falls back to gendered voices for characters without profiles

### Future Phase — Smart Collections
- Auto-populating collections based on rules (e.g., "All complete fics over 100k words tagged with Ship X")
- Dynamic filters saved as collection definitions

---

## Verification

1. **Source auth**: Enable a source, enter credentials, verify login succeeds. Enter wrong credentials — verify clean error and notification.
2. **Manual import**: Paste an AO3 URL, verify fic downloads with full metadata, HTML renders in reader, EPUB downloads correctly.
3. **Bookmark sync**: Connect AO3 account, trigger sync, verify bookmarks import with rec/private status and bookmark tags preserved.
4. **Update detection**: Archive a fic with ongoing chapters. Wait for or simulate an update. Verify new chapter downloaded, previous version stored in DB.
5. **Deleted fic handling**: Archive a fic, simulate source becoming unavailable. Verify badge appears, notification raised, fic still readable.
6. **Search**: Import 50+ fics. Verify full-text search returns relevant results. Verify all metadata filters work correctly.
7. **Favourites & tags**: Favourite a fic, create custom tags, apply via bulk operations. Verify filtered views work.
8. **Scanner integration**: Configure shared SQLite mode, add a tracked AO3 tag, verify discoveries appear in feed. One-click archive a discovery.
9. **Tag validation**: Add a tracked tag with an intentional typo — verify the UI catches it via autocomplete/validation.
10. **Print PDF**: Export a multi-chapter fic as A5 PDF. Verify title page, chapter breaks, margins, page numbers, running headers.
11. **TTS Basic**: Play a chapter with Web Speech API. Pause, close page, return — verify position resumed.
12. **TTS Gendered**: Generate gendered audio for a chapter with dialogue. Verify male/female voice switching on dialogue, narrator voice on narration.
13. **TTS Character**: Configure a voice profile (manual upload). Generate character voice audio. Verify correct voice used for that character's dialogue.
14. **GPU offload**: Configure GPU machine, take it offline, request character audio. Verify job queued, notification sent when machine returns and job completes.
15. **Notifications**: Trigger each notification type. Verify UI badge, in-app notification, and email delivery (when configured).
16. **Storage**: Fill audio cache past budget. Verify oldest non-favourite audio purged. Verify favourite audio retained.
17. **Auth expiry**: Invalidate a source's credentials. Verify auth_required notification with re-auth instructions.
18. **Cross-platform**: Deploy on Windows (IIS) and Linux (Docker). Verify identical functionality on both.
19. **Mobile**: Access archive and reader on mobile device. Verify responsive layout, readable text, functional controls.
20. **2FA**: Log in with correct password + wrong TOTP — verify rejection. Correct TOTP — verify access. Test "remember device" persistence.
