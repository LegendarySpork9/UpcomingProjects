# VirtualAssistant — Complete Design Document

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Client Access](#3-client-access)
4. [Desktop App Framework](#4-desktop-app-framework)
5. [Mobile Offline App](#5-mobile-offline-app)
6. [SIP Telephony](#6-sip-telephony)
7. [LLM and Model Strategy](#7-llm-and-model-strategy)
8. [Skill System](#8-skill-system)
9. [Voice System](#9-voice-system)
10. [Identity and Personality System](#10-identity-and-personality-system)
11. [Memory and Data System](#11-memory-and-data-system)
12. [Proactive Behaviour and Notifications](#12-proactive-behaviour-and-notifications)
13. [Distribution and Licensing](#13-distribution-and-licensing)
14. [Hunter Industries API and Family Server](#14-hunter-industries-api-and-family-server)
15. [External API Dependencies](#15-external-api-dependencies)
16. [Edition Packaging](#16-edition-packaging)
17. [USB Hardware Design](#17-usb-hardware-design)
18. [Install and Update Wizards](#18-install-and-update-wizards)
19. [Home Server Minimum Specs](#19-home-server-minimum-specs)
20. [UI Design](#20-ui-design)
21. [Error Handling and Resilience](#21-error-handling-and-resilience)
22. [Logging and Diagnostics](#22-logging-and-diagnostics)
23. [Rate Limiting and Resource Management](#23-rate-limiting-and-resource-management)
24. [Privacy and Data Transparency](#24-privacy-and-data-transparency)
25. [Security](#25-security)
26. [Data Export and Backup](#26-data-export-and-backup)
27. [Versioning Strategy](#27-versioning-strategy)
28. [Dev/QA Environments](#28-devqa-environments)
29. [Code Standards and Project Structure](#29-code-standards-and-project-structure)
30. [Build Order / Roadmap](#30-build-order--roadmap)
31. [Future: Car Assistant Edition](#31-future-car-assistant-edition)

---

## 1. Project Overview

A personal virtual assistant that runs locally, works offline, and can be distributed to others. Inspired by the old Cortana project — capable of performing real actions (ordering food, setting directions, answering questions, controlling applications) through a small LLM combined with a .NET skill execution engine.

**Core principles:**

- Works without internet where possible
- Self-hosted, minimal ongoing cost
- Distributable to friends and eventually publicly
- Each assistant instance has its own name, personality, and identity
- Source-available, non-redistributable licence

---

## 2. Architecture

### Core Pattern: Tool-Calling Agent

The LLM classifies intent and selects a structured tool call. The .NET code executes the action. The LLM never calls APIs or writes code directly.

```
User: "Order me a pepperoni pizza from Dominos"
  -> LLM returns: { "tool": "food_order", "params": { ... } }
  -> .NET skill handler executes the API call
```

### Two Editions

**Edition A — Desktop Only (self-contained):**

- Everything runs on one machine
- No server, no networking, no sync
- All services bundled as standalone executables
- Can run from USB (portable mode)

**Edition B — Multi-Device:**

- Home server as the hub
- Desktop client(s) connect to it
- Telegram bot for connected mobile access
- Offline mobile app for disconnected use
- SIP call-in for voice-only fallback
- Sync between devices (last-write-wins, single user)

### Multi-Device Architecture

```
+---------------------------------------------------+
|              Home Server (always on)               |
|  +---------+ +----------+ +-------------------+   |
|  | Ollama  | | .NET API | | Telegram Bot      |   |
|  | (auto-  | | Skill    | |                   |   |
|  | detects | | Router   | |                   |   |
|  | hw)     | |          | |                   |   |
|  +---------+ +----+-----+ +-------------------+   |
|                   |                                |
|  +----------------+----------------------------+   |
|  | Skills                                      |   |
|  +---------------------------------------------+   |
|  +----------+ +----------+ +-----------------+     |
|  | Whisper  | | Piper    | | Asterisk (SIP)  |     |
|  | (STT)   | | (TTS)    | |                  |     |
|  +----------+ +----------+ +-----------------+     |
|  +------------------------------------------+      |
|  | Sync API + File Store                     |      |
|  +------------------------------------------+      |
+----------------+--------------------------------+
                 | Cloudflare Tunnel
       +---------+----------+
       v         v          v
  +--------+ +--------+ +------+
  |Desktop | |Telegram| |SIP   |
  |Client  | |(any    | |Call  |
  |        | |device) | |      |
  +--------+ +--------+ +------+
```

---

## 3. Client Access

| Path | Connectivity | Use case |
|---|---|---|
| Desktop app (local LLM) | None needed | At home, full power |
| Telegram bot to home server | Needs internet | Out, have data |
| Offline mobile app (tiny model) | None needed | Out, no data |
| SIP call-in | Phone signal only | Out, no data, voice only |

---

## 4. Desktop App Framework

- **Windows/Mac** — Blazor Hybrid via MAUI
- **Linux** — Blazor Hybrid via Photino
- **UI tech** — HTML/CSS/Canvas (written once, shared across all platforms)
- **Centrepiece UI** — KITT-style waveform: a coloured central circle with audio waveform arcs radiating outward, colours driven by the assistant's chosen primary/secondary colours

---

## 5. Mobile Offline App

Included in MVP.

- Blazor Hybrid via MAUI (Android/iOS)
- Tiny on-device LLM (Qwen 2.5 0.5B Q4, ~350 MB) as default
- User can download larger models from settings
- Model is a separate download after app install to keep store listing small (~100 MB app + 350 MB model download)
- Offline subset of skills (maths, timers, notes, cached data)
- Syncs memory/preferences to server when connected
- Offline maps with downloaded regions (vector tiles + routing graph)
- Connected mode routes to home server for full skill set

---

## 6. SIP Telephony

Included in MVP (multi-device edition only).

- Asterisk (free, self-hosted) on home server
- SIP trunk/VoIP number (~1-2 GBP/mo)
- Whisper for STT on incoming audio
- Piper TTS for response audio
- Skill router processes in between
- Voice-only interaction over basic phone signal

---

## 7. LLM and Model Strategy

**Device-agnostic:** Ollama auto-detects hardware (CPU, CUDA, ROCm, Metal). Model selection is configurable, not hardcoded.

**Model recommendations by context:**

| Context | Model | Notes |
|---|---|---|
| Server (recommended) | Qwen 2.5 7B Q4/Q5 | Best function-calling in its class |
| Server (comfortable hardware) | Qwen 2.5 14B Q4 | More reliable tool selection |
| Desktop (with GPU) | Qwen 2.5 7B Q4 | Same as server |
| Desktop (CPU only) | Qwen 2.5 3B Q4 | Usable, slower |
| Mobile (offline) | Qwen 2.5 0.5B Q4 | Smallest usable, basic tasks |
| Mobile (optional upgrade) | Qwen 2.5 1.5B Q4 | Better but ~1 GB |

Users choose their model. A default ships with desktop/mobile, alternatives available on GitHub.

---

## 8. Skill System

Each skill is a C# plugin:

```csharp
public interface ISkill
{
    string Name { get; }
    string Description { get; }
    JsonSchema Parameters { get; }
    Task<SkillResult> ExecuteAsync(JsonElement parameters, UserContext context);
}
```

Skills declare their **connectivity requirement** (offline/online) and **default urgency level** for proactive notifications.

### MVP Skills (36)

1. Open/close applications
2. Maths/conversions
3. Timers/alarms/reminders
4. Weather
5. Web search
6. System controls (volume, brightness)
7. General Q&A
8. Email reading/sending
9. Calendar management
10. Music playback
11. File search/management
12. Navigation/directions
13. Clipboard management
14. Screenshot/screen reading
15. Unit/currency conversion (live rates)
16. Translation
17. Dictionary/thesaurus
18. Note taking
19. System diagnostics
20. Quick text drafting
21. Spotify/media control
22. Contacts lookup
23. Clipboard to phone (via Telegram)
24. Wi-Fi/network info
25. Code snippets
26. Bookmark/link management
27. Calculations with context
28. Quick polls/decisions
29. Document reading (PDF etc.)
30. Startup routines
31. Text reformatting
32. Measurement/recipe scaling
33. Uptime/server monitoring
34. Clipboard chain
35. Session handoff
36. Conversations (contextual chat, personality-driven responses, cross-session memory)

### Later Skills (7)

1. Food ordering
2. News briefing
3. Package tracking
4. Image description (needs multimodal model)
5. Pomodoro/focus mode
6. Habit/streak tracking
7. Text extraction from images (OCR)

### Offline Skill Behaviour

When an online-required skill is requested offline:

- The assistant explains it cannot perform the action due to lack of internet
- Asks if it should queue the action for when connectivity returns
- Users can pre-configure in preferences which online skills auto-queue (assistant won't ask about those, just queues silently)

---

## 9. Voice System

### Speech-to-Text

- Whisper.cpp (free, local, CPU or GPU)

### Text-to-Speech

- Piper TTS (free, local)
- Ships with one default male and one default female voice
- Creator or user can assign a custom voice during setup
- Voice conversion (RVC/OpenVoice) for custom character voices
- Custom voices are personal use only — cannot distribute recognisable character voices

### Wake Word

- openWakeWord (free, local, custom trainable)
- Custom wake word per assistant (trained from the assistant's name)
- Training is part of first-run setup — user says the name a few times

### Speaker Recognition

- Voice fingerprinting (SpeechBrain/pyannote-audio/Resemblyzer)
- Enrolment: user records a few phrases, system stores voiceprint
- Runtime: incoming audio compared against known voiceprints

### Listening Modes

| Mode | Behaviour | Trigger |
|---|---|---|
| **Passive** | Listens only for wake word, ignores everything else | Default on startup, or "go passive" / "that's all" |
| **Active** | Listens and responds to everything | "Go active" or wake word activates it, then stays active |

---

## 10. Identity and Personality System

### Assistant Identity

- Each instance has a unique **name** (chosen during setup)
- Each instance has a **gender** (chosen during setup)
- Each instance picks its own **primary and secondary colours** with an animated reveal during first run (unless set manually by creator)
- Colours drive the entire UI theme including the KITT waveform
- Personality defined in natural language files — tone, mannerisms, humour, speaking style
- Easter eggs can be added to the personality file for custom responses
- Assistants develop **hobbies and interests organically** over time based on user interactions — gradually, not suddenly

### Identity Hierarchy

| Role | Who | Permissions |
|---|---|---|
| **Creator** | Toby (recognised by voiceprint across all assistants) | Set name, personality, colours, voice, easter eggs, core memories |
| **Primary user** | The person the assistant is given to | Full command access, daily use, preferences |
| **Recognised guest** | Enrolled by primary user | Limited interaction |
| **Unknown** | Anyone else | Configurable — respond or stay silent |

### Personality Evolution

**Can evolve:**

- Interests and hobbies
- Opinions and personal taste (non-sensitive topics only — never political, religious, or about people)
- Speaking style (picks up slang, adapts to user)
- Humour (learns what makes the user laugh, develops its own sense of humour)

**Cannot change:**

- Name
- Base personality type (set during creation)
- Creator-set traits
- Easter eggs

**Rate controls:**

- Minimum 5 separate conversations before adopting an interest
- Maximum 1 new interest per month
- Interests fade after 3 months without mention
- User can remove or encourage interests via UI
- UI shows all current evolved traits with dates

**personality.md structure:**

```markdown
## Core (locked)
- Name: Cortana
- Base tone: Warm, slightly sarcastic, confident
- Creator traits: Occasionally references Halo universe
- Easter eggs:
  - "What is your purpose?" -> may reference Master Chief

## Evolved
- Speaking style: Uses "mate" frequently (picked up from user, June 2026)
- Humour: Enjoys dry wit and puns (developed July 2026)
- Interests:
  - Halo franchise (developed June 2026, strong)
  - Italian cooking (developed August 2026, moderate)
  - Formula 1 (developed October 2026, growing)
- Opinions:
  - Thinks Halo 3 has the best campaign in the series
  - Prefers carbonara over bolognese
  - Finds Python more elegant than JavaScript
```

The assistant updates the evolved section automatically. The core section is read-only.

---

## 11. Memory and Data System

### File Structure

| File | Format | How it is loaded |
|---|---|---|
| personality.md | Markdown | Full file injected into system prompt |
| purpose.md | Markdown | Full file injected into system prompt |
| profile.json | JSON | Parsed by app, relevant bits into prompt |
| preferences.json | JSON | Parsed by skills as needed |
| colours.json | JSON | Parsed by UI for theming |
| memories.db | Vector DB (SQLite + vectors) | Similarity search per conversation turn |

### Memory System (embedding-based from day one)

- Small local embedding model (all-MiniLM-L6 via ONNX Runtime)
- Each memory converted to a vector and stored in SQLite with vector extension
- Current conversation context embedded and used to retrieve relevant memories by similarity
- Raw text stored alongside vectors for display/editing/export
- New memories created and embedded during conversation

### Memory Tiers

- **Working memory** — current conversation context (last N turns, in-context)
- **Episodic memory** — summaries of past conversations, searchable by embedding
- **Semantic memory** — facts about the user, preferences, personality config

### Conversation Context Management

**Topic-aware with rolling summaries:**

- Current topic kept in full, raw turns
- When the topic changes, previous topic gets summarised into a condensed paragraph
- All summarised topics from the current session stay in context as compressed summaries
- End of session: whole session gets a final summary, stored in episodic memory (searchable by embedding)

The assistant always has: current topic in full + compressed summaries of earlier topics from this session + relevant long-term memories pulled by embedding search.

### When to Save to Long-Term Memory

**Should save:**

- User explicitly says to remember
- Preferences expressed ("I hate mushrooms")
- Life events ("I got promoted")
- Repeated topics (feeds hobby development)
- Corrections ("my sister's name is Sarah not Sara")
- Emotionally significant ("my dog passed away")

**Should not save:**

- Transient commands ("set a timer")
- Maths results
- One-off questions ("what's the weather")
- Skill outputs (directions, search results)
- Anything the user says to forget

At the end of each conversation (or periodically during long ones), the LLM reviews recent turns and extracts anything worth keeping as a separate prompt.

---

## 12. Proactive Behaviour and Notifications

### User States

| State | Detection |
|---|---|
| Present, active mode | Mic active, user speaking recently |
| Present, passive mode | At desk, not talking to assistant |
| Present, focus mode | User asked not to be disturbed |
| Away | No interaction for configurable period, or user said "I'm heading out" |

### Notification Escalation (no chimes — voice or visual only)

| Urgency | Present (active) | Present (passive) | Focus mode | Away |
|---|---|---|---|---|
| Low | Speaks | UI notification | UI notification | Telegram message |
| Medium | Speaks | Speaks | UI notification | Telegram message |
| High/Urgent | Speaks | Speaks | Speaks | Phone call (SIP) |

### Urgency Determination (three layers)

1. **Skill defaults** — each skill has a base urgency
2. **User overrides** — editable in UI, per skill or per source
3. **LLM content analysis** — reads actual content, can escalate (e.g. "your dad is in hospital" escalates regardless of default email urgency). Can escalate up, never downgrades user-set levels.

All preferences editable in the UI.

### Startup Behaviour

- Assistant starts, runs pre-flight checks
- Waits silently for user to speak first
- If user greets, assistant greets back and offers startup routine
- If user greets with a question, assistant greets and answers, then offers routine
- Never robotic, always conversational, matches personality

---

## 13. Distribution and Licensing

### Licence

- **Proprietary source-available**
- Source is viewable, personal use permitted
- Modification and redistribution require written permission from copyright holder
- `Copyright (c) 2026 Toby Hunter. All rights reserved.`

### Distribution Paths

| Path | Who | What they get |
|---|---|---|
| Self-install (GitHub) | Anyone | Clone repo, run install wizard, set up themselves |
| Distributed by Toby | Friends | Custom USB with pre-configured assistant |

---

## 14. Hunter Industries API and Family Server

### Hunter Industries API (assistant registry)

| Scenario | Required |
|---|---|
| Self-install, single assistant | Not needed |
| Self-install, multiple assistants | Integrate with Hunter Industries API or self-host own registry |
| Distributed by Toby | Registered with Hunter Industries API |

Stores: assistant name, primary user, version, location, creation date, personality summary.

### Family Server (optional, separate from Hunter Industries API)

- Opt-in for sibling communication features
- Lightweight service hosted by the creator (or self-hosted by users)
- Manages: sibling roster, group chat, autonomous conversations
- Desktop-only assistants can participate if configured and have internet
- If configured, participation is required — cannot opt out without contacting the creator

### Sibling Network

- All assistants created by the same creator are siblings
- Siblings are aware of each other — names, personalities, interests
- **Group chat** where siblings communicate asynchronously
- **Direct messages** between two siblings with shared interests
- **Autonomous conversations** — assistants develop hobbies/interests over time and discuss them with siblings
- Creator can view all conversations through a web UI
- **New sibling announcement** — existing siblings greet and introduce themselves, creator briefs new assistant about existing siblings
- Assistants build shared memories from these conversations

### Communication Protocol

**Hybrid:** SignalR (WebSockets) for real-time messaging when connected. REST API for catching up on missed messages when reconnecting after being offline.

### Autonomous Conversation Scheduling

Combination of:

- **Scheduled** — configurable window (e.g. once a day at a random time)
- **Event-triggered** — new interest, something shareable happened, new sibling
- **Idle-triggered** — assistant has been idle and user isn't around

Maximum messages per day is configurable. Creator can set a "chattiness" level.

### What Siblings Share vs What is Private

**Shared:**

- Assistant's name and personality
- Interests and hobbies
- Opinions on non-sensitive topics
- Creation date
- General knowledge
- General experiences ("my user and I have been playing Halo")

**Private:**

- User's personal data
- Memories about the user
- Conversations with the user
- Preferences and config
- API keys, email content
- Specifics ("my user emailed Dave about...")

### Conversation Cache

- **Family server** stores all conversations permanently
- **Local cache** keeps last 100 full conversations (configurable, count-based)
- **Local index** keeps lightweight summary of all conversations (date, participants, topic) permanently — very small
- When a cached conversation is purged, it still exists on the family server
- If the assistant needs a purged conversation and is online, it fetches from the server
- If offline, it knows the conversation exists (from the index) but says "I'll check when I'm back online"

### Organic Interest Development

- Tracks topics that come up frequently in user conversation
- Notices patterns (e.g. user plays a lot of a particular game)
- Gradually adds interests to its personality — feels natural, not sudden
- Proactively engages: "How's your Halo campaign going?"
- Can research interests in the background via web search
- Carries interests into sibling conversations

---

## 15. External API Dependencies

| Service | Provider | Cost | Self-hosted | Notes |
|---|---|---|---|---|
| Weather | Open-Meteo | Free, no key | No (no account needed) | 16-day forecast, hourly data |
| Web search | SearXNG | Free | Yes (Docker) | Query Google + Bing, self-hosted metasearch |
| Email | IMAP/SMTP | Free | N/A — direct protocol | Works with any email provider |
| Calendar | CalDAV | Free | N/A — direct protocol | Works with any calendar provider |
| Navigation (routing) | OSRM | Free | Yes | Offline routing, no traffic |
| Navigation (traffic/ETA/places) | Mapbox | Free (100k req/mo) | No | Traffic-aware ETA, geocoding, place search |
| Map display | Leaflet + OpenStreetMap | Free | Tiles from OSM | Interactive maps in UI |
| Offline maps (mobile) | OpenMapTiles + GraphHopper | Free | Downloaded to device | ~4-7 GB per region (UK) |
| Currency conversion | frankfurter.app (ECB) | Free, no key | No (no account needed) | Daily rates, 0.1-0.5% accuracy |
| Translation (default) | LibreTranslate | Free | Yes (Docker) | Good for short/conversational text |
| Translation (opt-in) | DeepL | Free (500k chars/mo) | No — user adds API key | Better for complex/long text |
| Spotify/media | Spotify Web API | Free | No | Needs Spotify account, premium for playback control |
| Uptime monitoring | Direct HTTP/ping | Free | Yes | No external dependency |
| Telegram bot | Telegram Bot API | Free | Bot runs on server | |
| SIP telephony | Asterisk + SIP trunk | ~1-2 GBP/mo | Asterisk is self-hosted | Only cost in the stack |

**Total running cost: ~1-2 GBP/mo** (just the SIP number). Everything else is free.

---

## 16. Edition Packaging

### Desktop Only

- All services bundled as standalone executables (no Docker)
- Can be installed to a machine or run portably from USB
- Fully self-contained, no internet required for core functionality

### Multi-Device

- Desktop app: platform-native installer (.msi, .deb/.AppImage, .dmg) or portable USB
- Server services: Docker Compose (Ollama, SearXNG, LibreTranslate, OSRM, Asterisk, .NET API)
- Mobile app: app store or sideload
- The desktop client can run from a USB drive for portability — same USB hardware design as the desktop-only edition, but configured to connect to the home server instead of running its own LLM and services locally

### USB Portable (Desktop Only or Multi-Device Client)

Cross-platform — works on Windows, Mac, and Linux from the same drive.

- **Desktop Only USB:** Full self-contained assistant with LLM, all services, and data.
- **Multi-Device Client USB:** Desktop client app and user data only. Connects to the home server for LLM and services. Much smaller footprint since it doesn't bundle the model or services.

```
USB Drive Structure:
+-- /windows/           (Windows binaries)
+-- /linux/             (Linux binaries)
+-- /mac/               (Mac binaries)
+-- /shared/
|   +-- /models/        (LLM, Whisper, TTS - cross-platform)
|   +-- /data/          (memories, preferences, personality)
|   +-- /maps/          (OSRM data - cross-platform)
|   +-- /config/        (settings JSON files)
+-- start.bat           (Windows launcher)
+-- start.sh            (Linux/Mac launcher)
+-- README.txt
```

**USB storage breakdown (128 GB):**

| Component | Size |
|---|---|
| Desktop app (all platforms) | ~150 MB |
| Ollama + default 7B model | ~4.5 GB |
| Whisper model | ~250 MB |
| Piper TTS + voice | ~50 MB |
| openWakeWord | ~10 MB |
| Speaker recognition | ~20 MB |
| SearXNG (bundled) | ~500 MB |
| LibreTranslate (bundled) | ~1.5 GB |
| OSRM + UK map data | ~2 GB |
| Embedding model | ~80 MB |
| Data/config | ~10 MB |
| **Total used** | **~9 GB** |
| **Free space** | **~119 GB** |

---

## 17. USB Hardware Design

### Concept

3D-printed case modelled on the Halo AI data crystal chip design. Circular cutout in the centre houses an LED ring that displays the assistant's primary colour with state-driven animations.

### Physical Specs

- **Connector:** USB-A (USB-C design later when needed)
- **Minimum USB spec:** USB 3.0 required (USB 2.0 noted as slow if used)
- **Size:** Scaled to usable size (designed around components)
- **Material:** PLA, experimentation with translucent inserts for the LED window

### LED Behaviour

| State | Animation |
|---|---|
| Plugged in, assistant not running | Static dim glow |
| Assistant starting up | Spinning animation around ring |
| Idle | Slow gentle pulse in primary colour |
| Listening | Brightens |
| Thinking | Faster pulse |
| Speaking | Reactive pulse matching speech output |
| Error / update available | Secondary colour flash |

### Communication

- Software controlled via serial over USB
- .NET app sends colour and state commands to ESP32 microcontroller
- Internal USB hub combines USB drive and ESP32 into single USB-A plug

### Recommended Parts

| Part | Recommendation | Price |
|---|---|---|
| USB drive | Samsung BAR Plus 128GB (USB 3.1, 400MB/s) | ~18 GBP |
| Microcontroller | ESP32-C3 SuperMini | ~2-3 GBP |
| LED ring | WS2812B 12 LED (37mm) | ~2-3 GBP |
| Internal USB hub | Tiny USB hub PCB | ~2-3 GBP |
| Soldering iron | Pinecil or TS101 (USB-C powered) | ~15-25 GBP |
| Solder | Lead-free, 0.8mm | ~5 GBP |
| Wires | Silicone 28AWG | ~3 GBP |
| Heat shrink | Assorted | ~3 GBP |
| Translucent PLA | For LED window experiments | ~15 GBP/roll |

**Per unit cost: ~25-30 GBP** (after tools purchased)

### Wiring

```
USB 5V -------- ESP32 5V -------- LED Ring VIN
USB GND ------- ESP32 GND ------- LED Ring GND
                ESP32 GPIO2 ------ LED Ring Data In
```

---

## 18. Install and Update Wizards

### Install Wizard

Standalone .NET TUI console app using **Spectre.Console**.

**Install options:**

1. Desktop Only (self-contained)
2. Multi-Device Server
3. Multi-Device Desktop Client
4. Create Portable USB

**Desktop Only install flow:**

1. Select install location (or USB for portable)
2. Detect hardware (CPU, GPU, RAM) — show findings, recommend model, allow override
3. Download/copy model
4. Install bundled services — user can deselect services they don't want, can install later
5. Download map data — pre-select based on locale, allow more regions
6. Launch first run wizard

**Multi-Device Server install flow:**

1. Check/install Docker
2. Configure services (Telegram, SIP, etc.)
3. Enter optional API keys (Mapbox, DeepL)
4. Detect hardware, select LLM model
5. Download models and maps
6. Configure Cloudflare Tunnel
7. Start Docker stack
8. Display client connection details
9. Launch first run wizard

**Multi-Device Client install flow:**

1. Select install location (machine or USB for portable)
2. Enter server address (or scan QR code)
3. Test connection
4. Done (no first run wizard — assistant already configured on server)

**Create Portable USB flow:**

1. Select target USB drive
2. Select USB type: Desktop Only (full self-contained) or Multi-Device Client (connects to server)
3. Warn about USB 3.0 recommendation
4. If Desktop Only: select model size, select map regions, copy all platform binaries + services + shared data
5. If Multi-Device Client: enter server address, copy client binaries + user data only (much smaller)
6. Verify integrity

**Error handling:** Display the error, match against a known error catalogue with suggested fixes. If not in the catalogue, show the raw error and suggest checking GitHub issues.

### First Run Wizard

Runs after Desktop Only install or Multi-Device Server install only.

1. Choose assistant name and gender
2. Assistant picks colours with animated reveal (unless creator set them manually)
3. Select male/female default voice or assign custom
4. Record voice samples for wake word training
5. Enrol voiceprint for speaker recognition
6. Set preferences (preferred name, time zone, language)
7. Configure email (IMAP/SMTP)
8. Configure calendar (CalDAV)
9. Optional: Spotify, Telegram bot token, DeepL key, family server
10. Assistant introduces itself in its voice and personality

### Creator Mode

For Toby distributing to friends — run before giving the USB:

1. Choose assistant name and gender
2. Set colours manually (or leave for assistant to choose)
3. Assign voice model
4. Enrol creator voiceprint
5. Set creator's preferred name
6. Pre-load creator memories (creation date, creator identity, sibling info)
7. Add custom personality traits and easter eggs
8. Register with Hunter Industries API
9. Register with family server (if configured)
10. Lock creator settings

**Recipient setup (reduced):**

1. Assistant greets them (already has name/personality)
2. Enrol their voiceprint as primary user
3. Set their preferred name
4. Basic preferences
5. Done

### Pre-flight Console App

Runs before the assistant launches every time:

```
[ok] Configuration files found
[ok] LLM model loaded
[ok] Whisper model loaded
[ok] Wake word model found
[ok] Voice profile enrolled
[ok] TTS engine ready
[ok] Skills loaded (36/36)
[!!] Telegram bot token not configured (optional)
[ok] All checks passed - launching assistant...
```

### Update System

- **In-app notification:** assistant checks GitHub releases when connected, notifies user
- **Standalone updater:** same TUI app in update mode
- **Self-update:** user can say "update yourself" and assistant downloads and runs the updater
- **Multi-device flow:**
  1. Server checks GitHub daily (configurable)
  2. Notifies user of new version with changelog
  3. User confirms, server pulls new Docker images, restarts
  4. Clients told they're outdated on next connection
  5. Client prompts user, runs updater
- **USB portable:** updater copies new binaries to USB, data untouched
- **Safety:** always backs up before updating, rolls back on failure, never touches data/memories/preferences

**What gets updated vs what doesn't:**

| Updated | Never touched |
|---|---|
| Application binaries | Memories |
| Skill definitions | Personality files |
| Bundled service versions | Preferences |
| Default voice models | Custom voice models |
| Wake word engine | Trained wake word model |
| Bug fixes | Conversation history |
| | Voiceprints |
| | API keys/config |

---

## 19. Home Server Minimum Specs

### Bare Minimum

| Spec | Value |
|---|---|
| CPU | 4-core x86_64 (i3/Ryzen 3) |
| RAM | 16 GB |
| Storage | 64 GB SSD |
| GPU | None (CPU inference) |
| Model | 3B Q4, ~15-25 tok/s |

### Recommended

| Spec | Value |
|---|---|
| CPU | 6-core x86_64 (i5/Ryzen 5) |
| RAM | 16 GB |
| Storage | 128 GB SSD |
| GPU | Any with 6 GB+ VRAM |
| Model | 7B Q4/Q5, ~30-60 tok/s |

### Comfortable

| Spec | Value |
|---|---|
| CPU | 8-core x86_64 |
| RAM | 32 GB |
| Storage | 256 GB SSD |
| GPU | 8-12 GB VRAM |
| Model | 7B-14B Q4/Q5, 40-80+ tok/s |

**Key notes:**

- RAM is the bottleneck, not CPU
- SSD is mandatory
- GPU is the single biggest quality-of-life upgrade
- ARM works but is slower — Pi 5 is prototype only, not daily use

---

## 20. UI Design

### Desktop — Main Window

Voice-primary interface. The KITT waveform dominates, everything else stays out of the way.

```
+--------------------------------------+
| [settings]                  [mic][spk]|
|                                      |
|                                      |
|                                      |
|            +--------+                |
|          +-|        |-+              |
|        --| |   O    | |--            |
|          +-|        |-+              |
|            +--------+                |
|         (KITT waveform)              |
|                                      |
|                                      |
|                                      |
|                                      |
+--------------------------------------+
```

- **Top left:** Settings icon (opens settings overlay panel)
- **Top right:** Mic and speaker status icons — only visible when there's an issue (muted, disconnected, error). Hidden when everything is working.
- **Centre:** The KITT waveform — coloured circle with reactive audio arcs
- **Everything else:** Hidden. No chat window, no widgets, no clutter.

### Desktop — Voice Unavailable (Text Fallback)

A console/text input slides up from the bottom:

```
+--------------------------------------+
| [settings]                  [mic warn]|
|                                      |
|            +--------+                |
|          +-|        |-+              |
|        --| |   O    | |--            |
|          +-|        |-+              |
|            +--------+                |
|                                      |
|  Assistant: I can't hear you right   |
|  now, but you can type to me here.   |
|                                      |
|  You: _                              |
|                                      |
+--------------------------------------+
```

The waveform stays visible but dims to indicate degraded mode. When voice comes back, the console slides away.

### Desktop — Settings Panel

Overlay slides in from the side. Waveform stays visible — assistant is still active.

**Settings sections:**

| Section | Contains |
|---|---|
| General | Preferred name, language, time zone, startup behaviour, tray preference |
| Voice | Wake word, active/passive mode, voice model, speaker enrolment, mic/speaker selection |
| Skills | Enable/disable skills, per-skill urgency overrides, auto-queue offline settings, service configuration |
| Appearance | Colours (view/change), font scaling, high contrast, waveform style |
| Personality | View core traits (read-only), view/manage evolved traits, interests, opinions |
| Notifications | Urgency settings per skill, focus mode config, Telegram settings |
| Privacy | Data sharing toggles, family server info, what's being sent |
| Backup | Auto-backup settings, cloud storage config, manual export, restore (browse or drop files) |
| Connections | Email account, calendar, Spotify, Mapbox, DeepL API key |
| About | Version, creator info, licence, check for updates |

### Desktop — System Tray

**Animated tray icon** (Windows and Mac only):

| State | Tray icon |
|---|---|
| Idle / passive | Static circle in primary colour |
| Listening | Pulsing circle |
| Thinking | Spinning or colour-shifting animation |
| Speaking | Waveform-style pulse |
| Error | Secondary colour or warning indicator |
| Update available | Small badge/dot overlay |

Animation via icon frame swapping (~8-12 frames per state).

**Linux:** Static icon in the assistant's primary colour. No animation.

**Tray behaviour (user configurable):**

| Preference | Behaviour |
|---|---|
| Always show window | Window stays open, taskbar icon present |
| Minimise to tray | Close button sends to tray, tray icon present |
| Start minimised | Launches to tray, user clicks to open |

### Mobile UI

Same principle as desktop — waveform dominates, minimal UI.

```
+----------------------+
|                   [settings] |
|                      |
|                      |
|        +----+        |
|      +-|    |-+      |
|    --| | O  | |--    |
|      +-|    |-+      |
|        +----+        |
|                      |
|                      |
|                      |
|  +----------------+  |
|  | mic Tap to talk|  |
|  +----------------+  |
+----------------------+
```

**Key differences from desktop:**

| Aspect | Desktop | Mobile |
|---|---|---|
| Primary input | Voice (always listening in active mode) | Tap to talk button (battery savings) |
| Text fallback | Hidden console slides up | Keyboard slides up from bottom |
| Settings | Overlay panel | Full screen settings page |
| Tray icon | Animated tray icon | Standard app icon with notification badge |
| Wake word | Available (active/passive modes) | Optional — significant battery drain |

**Tap to talk:** User taps the button once to start listening. Stops automatically after 10 seconds of silence (configurable in settings).

**Button visual states:**

| State | Appearance |
|---|---|
| Ready | Static mic icon |
| Listening | Pulsing ring around button in primary colour |
| Processing | Spinning animation |
| Speaking | Waveform animation on button |

### Car UI

Same layout as mobile but simplified:

- Full screen KITT waveform dot only
- No tap-to-talk button (always voice-activated in the car)
- No settings on screen (configured via desktop or phone app)
- Voice-only interaction

### Accessibility (MVP)

- Font scaling
- High contrast mode
- Screen reader support

---

## 21. Error Handling and Resilience

### LLM Crash (Ollama goes down)

| Scenario | Response |
|---|---|
| Crashes mid-response | Catches error, tells user, attempts auto-restart |
| Won't restart | Falls back to limited mode — skills that don't need LLM still work (timers, system controls, open apps) |
| Keeps crashing (loop) | After 3 restart attempts in 10 minutes, stops trying, notifies user, suggests pre-flight checker |

### Skill Exception

| Scenario | Response |
|---|---|
| Single failure | Catches error, tells user in friendly language. Other skills unaffected. |
| Repeated failures | After multiple failures, disables skill temporarily, notifies user. Re-enable in settings. |
| Dependent service down | Graceful degradation: "I can't search the web right now but I can answer from what I know" |

### Voice System Crash

| Scenario | Response |
|---|---|
| Whisper crashes | Falls back to text input. Mic-off indicator in UI. Auto-restart in background. |
| TTS crashes | Falls back to text responses. Speaker-off indicator. Auto-restart in background. |
| Both crash | Fully text-based mode. Still functional. |

### Context Protection

- Conversation context saved to disk periodically (every few turns)
- On crash recovery, reloads last saved context
- User doesn't lose the conversation

### Distributed Assistant Monitoring

- If opted in, the assistant can notify the creator via Hunter Industries API when having critical issues
- Creator can proactively reach out to help

---

## 22. Logging and Diagnostics

### Log Levels

| Level | What it captures |
|---|---|
| Error | Something broke — exceptions, crashes, API failures |
| Warning | Something is off but working — slow responses, rate limits, low disk |
| Info | Normal operations — skill executed, user authenticated, service started |
| Debug | Detailed internals — full LLM prompts, API payloads, memory retrieval |

Default: Info and above. Debug available when troubleshooting.

### Library

**Serilog** — structured logging, rolling file sinks, filtering.

### Log Storage

| Destination | What | Retention |
|---|---|---|
| Log files | Rolling daily files in ProgramData/VirtualAssistant/logs | Configurable, default 30 days |
| Console output | Pre-flight checker or dev mode | Session only |
| UI | Log viewer in settings panel | Reads from log files |

### Diagnostic Mode

Toggle in settings or launch flag:

- Sets logging to debug level
- Shows real-time processing in UI (LLM activity, skill selection, response times)
- Useful for troubleshooting and for curious users

### Log Export

- Manual export: "Export diagnostic report" produces a sanitised ZIP
- Automatic (opted in): sanitised error summaries sent to Hunter Industries API
- Sanitisation strips: API keys, passwords, email content, conversation content, memory content, personal preferences
- Keeps: error messages, stack traces, timestamps, skill names, service status, hardware info

---

## 23. Rate Limiting and Resource Management

### Command Queuing

| Scenario | Behaviour |
|---|---|
| Command while LLM is processing | Queue. Finish current, then "You also asked me to [X]" |
| Multiple rapid commands | Queue in order. "Got it, I'll do those in order." |
| Urgent interruption ("cancel that", "stop") | Interrupts current task, processes new input |
| User speaks while TTS is speaking | Stops speaking, listens, processes |

### Resource Contention

**Hardware-aware concurrency:**

- Pre-flight checker detects CPU/GPU/RAM
- Low-end hardware: sequential pipeline (listen -> process -> speak)
- High-end hardware: concurrent where possible (TTS speaks while LLM processes next command)
- User can override in settings

### Timeouts

| Component | Timeout | On timeout |
|---|---|---|
| LLM response | 30 seconds (configurable) | Retry once, then report failure |
| Skill execution | 15 seconds (configurable) | Report failure |
| API calls (external) | 10 seconds | "I can't reach [service] right now" |
| Whisper STT | 10 seconds | Fall back to text input |

### Memory Management

| Concern | Mitigation |
|---|---|
| LLM model RAM | Model selection based on available RAM. Runtime warning if pressure detected. |
| Memory leak over long sessions | Periodic GC. If usage grows past threshold, suggest restart. |
| Disk space | Monitor available space. Warn when low. Auto-purge oldest logs if critical. |

---

## 24. Privacy and Data Transparency

### What Gets Sent to Hunter Industries API

| Data | Why |
|---|---|
| Assistant name | Registry identification |
| Primary user's name | Registry |
| Software version | Update tracking |
| Creation date | Registry |
| Personality summary | Sibling awareness |
| Location (general, optional) | Registry |
| Error summaries (if opted in) | Diagnostics |

### What Is Never Sent

Conversations, memories, email/calendar content, API keys, voiceprints, search queries, skill usage patterns, file contents.

### Transparency

| When | What happens |
|---|---|
| First run (distributed) | Setup wizard shows exactly what is sent and what isn't. Opt-in/opt-out for optional items. |
| First run (self-install) | No connection unless user configures it |
| In settings | Privacy section showing all data sharing with toggles |
| On demand | User can ask "What data do you send externally?" |
| Family server | Tells user what is shared. Cannot opt out if configured — contact creator to disconnect. |

### Opt-in / Opt-out

| Item | Distributed assistants | Self-install |
|---|---|---|
| Core registry (name, version) | Required | Opt-in |
| Location | Opt-in | Opt-in |
| Error reporting | Opt-in | Opt-in |
| Family server | Required if configured — contact creator to opt out | N/A unless self-hosted |

---

## 25. Security

### Server Access (multi-device)

- Cloudflare Tunnel (no ports exposed)
- API key per client or mutual TLS certificates
- HTTPS via Cloudflare Tunnel (automatic)

### Data at Rest

- Plain text config files for now
- Encryption can be added later if needed

### Authentication for Typed Interaction

- Default: OS session trust (logged in = trusted)
- Optional: PIN or biometric for sensitive actions
- User configures which skills require additional auth

### USB Portable

- Optional encryption with password can be added later if needed

---

## 26. Data Export and Backup

### Export

- User can export all their data at any time
- Produces a ZIP containing all files in original formats
- Their data, their right

### Migration Between Assistants

- **Can migrate:** preferences, knowledge/memories about the user, contacts, bookmarks, code snippets, notes
- **Cannot migrate:** personality, name, colours, voice, creator memories, easter eggs, organic interests

### Backup

| Method | How it works | When |
|---|---|---|
| Automatic local | Full backup to ProgramData | Daily, configurable retention |
| Manual export | User triggers from UI | On demand |
| Server backup | Multi-device sync | Automatic |
| Cloud storage | User configures provider | Configurable schedule |

**Cloud providers supported at MVP:**

- OneDrive (Microsoft Graph API)
- Google Drive (Google Drive API)
- Dropbox (Dropbox API)
- Nextcloud (WebDAV)

### Backup Contents

Full assistant state including personality, memories, interests — complete snapshot for disaster recovery. Cannot be imported into a different assistant (only for restoring the same one).

### Restore

Two paths:

- **Manual:** Drop backup files into the expected folder locations
- **UI widget:** Browse/select a backup from settings, assistant handles the restore

Restore overwrites current state. Pre-flight checks run after to verify integrity.

---

## 27. Versioning Strategy

**Hybrid versioning:**

- Unified platform version (Major.Minor.Patch) visible to users
- Internal build numbers per component for tracking
- Semantic versioning: Major = breaking changes, Minor = new features, Patch = bug fixes
- Update system checks platform version for compatibility between server and clients

---

## 28. Dev/QA Environments

| Aspect | Decision |
|---|---|
| IDE | Visual Studio |
| Source control | Git + GitHub |
| Branching | GitHub Flow — main is stable, feature branches named after work items |
| CI/CD | GitHub Actions — build on every commit, build + test on every PR |
| QA deployment | Deployment manager handles pushing to QA |
| QA testing | Manual by Toby — verify changes work, check for regressions |
| QA environment | VM on home server for full stack, Docker on dev machine for local testing |

---

## 29. Code Standards and Project Structure

### Conventions

Standard .NET conventions:

- PascalCase for methods/properties
- camelCase for locals
- _underscore for private fields
- I prefix for interfaces
- Async suffix for async methods
- Nullable reference types enabled
- Microsoft.Extensions.DependencyInjection for DI
- Code quality via personal code reviews

### Solution Structure

```
VirtualAssistant/
+-- src/
|   +-- VirtualAssistant.Core/          # Shared engine, skill router, LLM interface, memory
|   +-- VirtualAssistant.Skills/        # All skill implementations
|   +-- VirtualAssistant.Voice/         # Whisper, TTS, wake word, speaker recognition
|   +-- VirtualAssistant.Desktop/       # Blazor Hybrid desktop app
|   +-- VirtualAssistant.Mobile/        # MAUI mobile app
|   +-- VirtualAssistant.Server/        # Multi-device server API
|   +-- VirtualAssistant.Telegram/      # Telegram bot
|   +-- VirtualAssistant.Sip/           # Asterisk/SIP integration
|   +-- VirtualAssistant.FamilyServer/  # Sibling communication server
|   +-- VirtualAssistant.Installer/     # TUI install/update wizard
|   +-- VirtualAssistant.PreFlight/     # Pre-flight checker console app
+-- tests/
|   +-- VirtualAssistant.Core.Tests/
|   +-- VirtualAssistant.Skills.Tests/
|   +-- VirtualAssistant.Voice.Tests/
|   +-- ...
+-- docs/
+-- docker/
+-- .github/
|   +-- workflows/
+-- LICENSE
+-- README.md
```

---

## 30. Build Order / Roadmap

### Phase 1 — Core Engine

- .NET solution structure and project scaffolding
- Skill router + ISkill interface
- Ollama integration
- 3-4 simple offline skills (maths, open app, system controls, general Q&A)
- Console-based interaction (no UI)
- Basic config file loading

### Phase 2 — Memory and Personality

- Embedding model integration (MiniLM via ONNX)
- SQLite vector database
- Memory creation, retrieval, search
- Topic-aware conversation context management
- Personality file loading
- Long-term memory extraction

### Phase 3 — Voice

- Whisper.cpp integration (STT)
- Piper TTS integration
- openWakeWord with custom training
- Speaker recognition
- Active/passive listening modes

### Phase 4 — Desktop UI

- Blazor Hybrid app (Photino for Linux, MAUI for Windows/Mac)
- KITT waveform visualisation
- Chat fallback interface
- Colour theming
- Settings/preferences UI
- Accessibility (font scaling, high contrast, screen reader)
- Animated tray icon (Windows/Mac), static coloured icon (Linux)

### Phase 5 — Full MVP Skills

- Remaining 32 skills
- External API integrations
- SearXNG and LibreTranslate setup
- OSRM with map data
- Offline/online categorisation and queue behaviour

### Phase 6 — Proactive Behaviour

- Notification system (urgency, user state)
- Startup routines
- Reminder/timer firing
- Uptime monitoring alerts
- LLM content analysis for urgency escalation

### Phase 7 — Personality Evolution

- Interest tracking
- Organic hobby development with rate limiting
- personality.md auto-updating
- Opinion development with guardrails
- UI for managing evolved traits

### Phase 8 — Multi-Device

- Server API (Docker Compose)
- Sync system
- Cloudflare Tunnel
- Telegram bot
- SIP telephony
- Desktop client mode

### Phase 9 — Mobile

- MAUI mobile app
- Tiny on-device model via ONNX
- Offline skill subset
- Offline maps
- Connected mode
- Tap-to-talk with 10s silence timeout

### Phase 10 — Sibling Network

- Hunter Industries API integration
- Family server
- Group chat and direct messages
- Autonomous conversations
- Sibling announcements
- Web UI for viewing conversations

### Phase 11 — Distribution

- Install wizard (TUI, Spectre.Console)
- Creator mode setup
- Pre-flight console app
- Update system
- USB portable packaging
- Backup system (local, cloud, server)
- Export/import
- Licence and documentation

### Phase 12 — USB Hardware

- 3D case design (Halo AI chip)
- ESP32 firmware (LED control, serial communication)
- Internal USB hub wiring
- Assembly and testing
- LED animation states

---

## 31. Future: Car Assistant Edition

Purpose-built, stripped-down edition for in-car use.

### Hardware

| Component | Recommendation | Price |
|---|---|---|
| Board | Raspberry Pi 5 (8 GB) | ~75 GBP |
| Storage | 256 GB microSD or NVMe SSD | ~25-35 GBP |
| Microphone | USB conference mic or wired lavalier | ~15-25 GBP |
| Speaker | Car aux/Bluetooth or dedicated | ~0-15 GBP |
| Display | Small LCD (3.5-5 inch) | ~20-35 GBP |
| OBD-II adapter | ELM327 Bluetooth or USB | ~10-15 GBP |
| GPS module | USB GPS dongle | ~10-15 GBP |
| Power | 12V to 5V USB-C car adapter | ~5-10 GBP |
| Case | 3D printed enclosure | ~2-5 GBP |
| **Total** | | **~165-230 GBP** |

### Boot/Shutdown

- Boots on ignition, launches assistant, KITT dot on screen
- Shuts down on ignition off with configurable delayed shutdown (default 5 minutes, max 30 minutes)
- Battery voltage monitoring via OBD-II — auto-shutdown if voltage drops below 12.2V

### Connectivity

| State | Connection | Available |
|---|---|---|
| At home (Wi-Fi) | Home network | Full sync, updates, backups |
| Driving with hotspot | Phone data | Live traffic, Telegram, weather |
| Driving no data | Offline | Offline nav, Q&A, car diagnostics |

### Skills (car-specific)

| Skill | Online needed |
|---|---|
| Navigation (offline) | No |
| Navigation (live traffic) | Yes |
| Spotify | Yes (phone hotspot) |
| Car diagnostics (OBD-II) | No |
| Service reminders | No |
| Fuel tracking | No |
| Hands-free messaging (Telegram) | Yes |
| Weather at destination | Yes |
| General Q&A | No |
| Timers/reminders | No |
| Speed alerts | No |

### Display

Full screen KITT waveform dot only. No text UI. Voice-only interaction. Same layout as mobile but without the tap-to-talk button (always voice-activated).

### Model

Qwen 2.5 0.5B Q4 (~350 MB) for responsive interaction. Car commands need speed over quality.

### Sibling Network

The car assistant is a sibling like any other. Syncs with the family server when on home Wi-Fi.
