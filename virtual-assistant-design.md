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
19. [Creator's Workbench](#19-creators-workbench)
20. [Home Server Minimum Specs](#20-home-server-minimum-specs)
21. [UI Design](#21-ui-design)
22. [Error Handling and Resilience](#22-error-handling-and-resilience)
23. [Logging and Diagnostics](#23-logging-and-diagnostics)
24. [Rate Limiting and Resource Management](#24-rate-limiting-and-resource-management)
25. [Privacy and Data Transparency](#25-privacy-and-data-transparency)
26. [Security](#26-security)
27. [Data Export and Backup](#27-data-export-and-backup)
28. [Versioning Strategy](#28-versioning-strategy)
29. [Dev/QA Environments](#29-devqa-environments)
30. [Code Standards and Project Structure](#30-code-standards-and-project-structure)
31. [Build Order / Roadmap](#31-build-order--roadmap)
32. [Future: Car Assistant Edition](#32-future-car-assistant-edition)
33. [Future: Droid Assistant Edition](#33-future-droid-assistant-edition)
34. [Addon: Hologram Display](#34-addon-hologram-display)
35. [Addon: Smart Glasses](#35-addon-smart-glasses)

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
- All services bundled as standalone executables
- Can run from USB (portable mode)
- Optional: connect to a home server for LLM processing offload (toggle in settings, default off). When enabled, LLM inference is routed to the server if reachable, with automatic fallback to local processing if unavailable
- Optional: participate in the sibling network (family server) when internet is available
- Does not gain Multi-Device services (Telegram bot, SIP, mobile sync, multi-device clients)

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

### Cloud LLM Option (discouraged)

For users who cannot run a local LLM due to technical or financial constraints — insufficient hardware and no access to a home server — the assistant supports routing LLM inference to a cloud-based model API as an alternative to local processing.

**⚠ This option is strongly discouraged.** One of the core principles of this project is that it works without internet and keeps all data local. Using a cloud LLM **violates this principle.** The system prompt — including the assistant's personality, the user's relevant memories, conversation history, and preferences — is sent to a third-party API on every request. The user loses the privacy guarantees that the local-first design exists to provide. This option exists solely as a concession for users who want the assistant but genuinely cannot meet the hardware requirements by any other means.

During setup, if the user selects cloud LLM, the installer presents a clear disclaimer:

> **You are choosing to route your assistant's intelligence through a third-party cloud service. This means:**
>
> - Your conversations, including your assistant's personality and your personal memories, will be sent to an external server on every interaction
> - You lose the privacy and data ownership guarantees that are a core principle of this project
> - You will incur ongoing API costs billed by the cloud provider
> - The assistant will not function without an internet connection — there is no offline fallback for LLM processing
>
> **This project is designed to run locally. Cloud LLM is a last resort, not a recommendation.**

The user must explicitly acknowledge this before proceeding.

**How it works:**

The LLM interface in the .NET skill router is endpoint-agnostic. It sends messages and receives completions with tool calls. The change is the endpoint URL and authentication — nothing else in the stack (skills, memory, personality, voice, wake word) is affected.

| Provider | Endpoint | Authentication | Notes |
|---|---|---|---|
| Local (Ollama) | `http://localhost:11434` | None | Default, recommended |
| Home server (Ollama remote) | Server address via Cloudflare Tunnel | API key or mTLS | Requires Processing Server or Multi-Device Server |
| Anthropic | `https://api.anthropic.com` | User's API key | Claude models. Function calling supported. |
| OpenAI | `https://api.openai.com` | User's API key | GPT models. Function calling supported. |
| OpenAI-compatible | Custom URL | User's API key | Covers any compatible endpoint (e.g. rented GPU running Ollama remotely) |

The user provides their own API key. The project does not ship with, subsidise, or manage cloud API access. Configuration is in the Connections section of settings.

**When cloud LLM is active**, a persistent indicator is shown in the settings panel and optionally on the desktop UI — a clear reminder that conversations are leaving the device.

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
| Low | Speaks | UI notification | Queued | Telegram message |
| Medium | Speaks | Speaks | UI notification | Telegram message |
| High | Speaks | Speaks | Speaks | Phone call (SIP) |
| Critical | Speaks immediately | Speaks immediately | Overrides focus mode, speaks | Phone call (SIP) + repeated |

Critical urgency is reserved for genuine emergencies. It overrides focus mode unconditionally. The LLM content analysis layer can escalate to Critical (e.g. "your dad is in hospital"). Users can also mark specific skills or sources as Critical.

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

### Legal Considerations for Custom Content

#### Custom 3D Models (Hologram Addon)

Users may upload 3D avatar models for the hologram display, including models of copyrighted characters (e.g. Cortana from Halo, anime characters, game characters).

**Project responsibility:**

- The project ships with an original default avatar (3D KITT waveform) — no third-party IP
- Custom model uploads are entirely the user's responsibility
- During model upload, a one-time acknowledgement is required: the user confirms they have the right to use the model
- VRM format includes built-in licence metadata (allowed usage, commercial use, redistribution) — the upload pipeline reads this and warns if the model's own metadata says redistribution isn't permitted

**Creator distribution responsibility:**

- If a creator distributes their assistant to others, the setup wizard flags custom models specifically
- The creator must confirm they have distribution rights for any included custom avatar model
- Distributed versions should only include original or appropriately licensed content

**User private use (extracted game/media assets):**

- Extracting models from games typically violates the game's EULA (reverse engineering/asset extraction is prohibited by most publishers including Microsoft)
- May violate DMCA anti-circumvention provisions if files are encrypted or DRM-protected
- A fair use argument for purely private, non-commercial, transformative use is plausible but untested in this specific context
- No rights holder has ever pursued legal action against an individual for privately using extracted assets without distribution
- Practical enforcement risk for purely private use is effectively nil
- Risk increases significantly with any form of distribution, streaming, or commercial use

#### Voice and TTS Models

**Using original voice recordings from games/media:**

- Copyrighted by the publisher — same grey area as model extraction
- Voice actors have additional right of publicity protections over their voice (stronger in the US under California/Tennessee law, weaker in the UK)
- Emerging legislation (Tennessee ELVIS Act, California AB 2602/AB 1836, proposed federal NO FAKES Act) strengthens voice performance protections
- Using original recordings as training data for AI/TTS is legally riskier than simple playback

**AI voice cloning of real actors:**

- Legally risky under right of publicity laws, even for personal use
- Landmark cases (Midler v. Ford 1988, Waits v. Frito-Lay 1992) established that deliberately imitating a distinctive, widely-known voice to sell a product is actionable
- Not recommended for this project

**Commissioning original voice performances (recommended approach):**

- Hiring a voice actor to perform in the style/personality of a fictional character is legally defensible
- Direct the actor toward the character's style and tone, not to clone the specific original actor
- A voice itself is not copyrightable — copyright protects fixed recordings, not vocal qualities
- Right of publicity protects real people, not fictional characters
- The resulting TTS voice is an original performance by a different person

**Contract requirements for commissioned voice work:**

- Work-for-hire clause — commissioner owns the recordings outright
- Explicit AI/TTS training consent — separate from the recording licence, clearly stating the recordings will be used to train a text-to-speech model
- Defined scope of TTS model use
- Separate compensation for AI training rights (industry range: £1,000-£10,000 for limited grants)
- Redistribution terms — if the TTS voice ships with distributed versions, the contract must explicitly allow this
- Revocation rights and data purging timeline

**What the project must never do:**

- Never include instructions for extracting copyrighted assets from games or media
- Never ship copyrighted content from third parties
- Never present custom content as officially affiliated with any IP holder
- Never use trademarked character names (e.g. "KITT", "Cortana", "Knight Rider") in distributed product branding
- Never use copyrighted dialogue from source material as recording scripts — write original lines

#### UK vs US Legal Context

- UK law does not recognise a standalone right of publicity — the primary recourse is "passing off" which requires proving implied endorsement and resulting damage
- UK protections for voice actors are currently weaker than the US, though the "Save Our Voices Now" campaign (2026) is pushing for statutory voice protections
- The project is UK-based — UK law applies to the project itself, but users in other jurisdictions are subject to their local laws
- The README should note that legal considerations vary by jurisdiction and users should check their local laws

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
| Navigation (primary, online) | Google Maps Platform | Free (see below) | No — user adds API key | Navigation SDK, Places API, Routes API, Geocoding API. Primary online navigation provider. |
| Navigation (offline fallback) | OSRM | Free | Yes | Offline routing with downloaded map data. No traffic, no re-routing, limited POI database. |
| Navigation (geocoding fallback) | Mapbox | Free (100k req/mo) | No | Fallback geocoding and place search when Google Maps is unavailable |
| Map display | Leaflet + OpenStreetMap | Free | Tiles from OSM | Interactive maps in UI |
| Offline maps (mobile) | OpenMapTiles + GraphHopper | Free | Downloaded to device | ~4-7 GB per region (UK) |
| Fuel prices | UK Government Fuel Finder | Free | No — GOV.UK One Login (OAuth 2.0) | REST API + CSV. Mandatory station reporting within 30 min of price changes. Prices, locations, brands, amenities, opening hours. |
| Currency conversion | frankfurter.app (ECB) | Free, no key | No (no account needed) | Daily rates, 0.1-0.5% accuracy |
| Translation (default) | LibreTranslate | Free | Yes (Docker) | Good for short/conversational text |
| Translation (opt-in) | DeepL | Free (500k chars/mo) | No — user adds API key | Better for complex/long text |
| Spotify/media | Spotify Web API | Free | No | Needs Spotify account, premium for playback control |
| Uptime monitoring | Direct HTTP/ping | Free | Yes | No external dependency |
| Telegram bot | Telegram Bot API | Free | Bot runs on server | |
| SIP telephony | Asterisk + SIP trunk | ~1-2 GBP/mo | Asterisk is self-hosted | Only cost in the stack |

**Total running cost: ~1-2 GBP/mo** (just the SIP number). Everything else is free.

### Navigation — Google Maps Platform

Google Maps is the primary online navigation provider. Self-hosted alternatives (OSRM, Mapbox) are retained as offline fallbacks but have significant limitations: no real-time traffic, no automatic re-routing, and a smaller POI database. Google Maps provides the superior experience when online.

**Google Maps free tiers (per-API monthly allowances, not pooled):**

| API | Free/Month | Per 1,000 After Free | Used For |
|---|---|---|---|
| Navigation SDK | 1,000 destinations | £21 | Real-time turn-by-turn on mobile (headless mode). Handles re-routing, traffic, ETA — all on-device after initial route fetch. |
| Places API (Nearby Search) | 10,000 | £4.20 | "Nearest petrol station", "nearest Aldi" — find places by type or name |
| Routes API (Essentials) | 10,000 | £4.20 | Route calculation for desktop (no Navigation SDK on desktop) |
| Routes API (Pro) | 5,000 | £8.40 | Traffic-aware routing for desktop |
| Geocoding API | 10,000 | £4.20 | Address to coordinates and vice versa |

A typical navigation request = 1 Places API call + 1 Navigation SDK destination. At 15 requests/day = ~450/month — well within all free tiers. Multi-destination routes (e.g. petrol station then holiday cottage) count as 1 Navigation SDK destination with waypoints.

**How navigation works on mobile:**

1. User requests navigation ("Take me to the cheapest diesel, then to the holiday cottage in Devon")
2. The assistant's LLM classifies the intent and identifies destinations, travel mode, and any criteria (cheapest diesel)
3. The assistant calls the relevant APIs — Places API for place search, Fuel Finder API for fuel prices if needed, Geocoding API for address resolution
4. The assistant presents the options and confirms with the user
5. On confirmation, the assistant launches the **Google Maps app on the phone** with the destination(s) and route pre-loaded via Android intent (`google.navigation:`) or iOS URL scheme (`comgooglemaps://`)
6. Google Maps handles the full navigation UI, voice guidance, traffic, and re-routing on the phone
7. Simultaneously, the **Navigation SDK runs in headless mode** in the assistant's mobile app, providing a real-time turn-by-turn data feed — manoeuvre type, road names, distance, ETA, lane guidance — all calculated on-device from the initial route fetch with no additional API calls during navigation
8. The glasses HUD receives this data feed over BLE and displays arrows, street names, distance, and ETA
9. If the phone is connected to the car via **Android Auto or CarPlay**, the assistant asks: "Shall I show navigation on your car's display or keep it on your phone?" If the user chooses the car, Google Maps projects to the car screen
10. If the car supports direct satnav integration (e.g. Tesla Fleet API), the assistant offers to send the destination to the car's built-in navigation instead

**Multi-destination routes:**

The assistant supports multi-stop routes. When the user requests multiple destinations ("petrol station, then the holiday cottage"), the assistant:

1. Resolves all destinations (place search, fuel prices, geocoding)
2. Presents the full route: "I'll take you to the Shell on the A38 for diesel at 134.9p — that's 5 minutes away — then on to the cottage in Devon, about 2 hours 15 from there. Sound good?"
3. On confirmation, launches Google Maps with the route including waypoints (intermediate stops) and the final destination
4. The Navigation SDK tracks progress through each waypoint, updating the glasses HUD as each leg completes

**How navigation works on desktop:**

The desktop does not have the Google Maps app or the Navigation SDK. Instead:

1. The assistant calls the **Routes API** to calculate the route
2. The route is displayed on an interactive map (Leaflet + OpenStreetMap tiles) in the desktop UI
3. If glasses are connected to the desktop, the route data is pushed to the glasses HUD for pre-departure review
4. For actual turn-by-turn guidance, the assistant suggests transferring the route to the phone: "I've planned your route. Shall I send it to your phone for navigation?"
5. The route is sent to the mobile app, which launches Google Maps and starts the Navigation SDK as described above

**Car satnav integration (manufacturer-dependent):**

Direct integration with car built-in navigation is available where manufacturer APIs permit. This is not a guaranteed feature — it depends on the car.

| Manufacturer | Integration | Status |
|---|---|---|
| Tesla | Fleet API — `navigation_request` sends destinations directly to the car's nav | Fully supported |
| Volvo (2026+ AAOS) | Google Maps "Send to Vehicle" via same Google account on phone and car | Best-effort — consumer feature, not a programmable API |
| Other manufacturers | Added as APIs become available | Not supported at launch |

Android Auto and CarPlay are the universal fallback — the phone's Google Maps navigation projects to the car screen regardless of manufacturer.

**Offline navigation (OSRM fallback):**

When the phone has no internet connection, the assistant falls back to OSRM with downloaded map data. Limitations are documented clearly to the user:

- No real-time traffic data
- No automatic re-routing if the user deviates
- Limited POI database (cannot search for "nearest Aldi" — only works with coordinates or pre-cached locations)
- No fuel price data (requires internet)
- Route quality may be lower than Google Maps

The assistant informs the user when falling back: "I'm offline, so I'll use downloaded maps. I won't have traffic data or re-routing."

### Fuel Prices — UK Government Fuel Finder

The Fuel Finder API provides real-time UK fuel prices as open data, mandated by the Motor Fuel Price (Open Data) Regulations 2025. Operated by VE3 Global, stations must report price changes within 30 minutes.

**Data available:** Current retail prices by fuel type (petrol, diesel), station addresses, operators/brands, amenities, and opening hours.

**Authentication:** OAuth 2.0 via GOV.UK One Login. Free access for developers and individuals.

**Also available as:** CSV download (updated twice daily) and email subscription — useful as a backup if the API is unavailable.

**How the assistant uses it:** When the user requests fuel-related navigation ("cheapest diesel nearby"), the assistant queries Fuel Finder for stations within a radius of the user's location, compares diesel prices, and presents the cheapest option(s) before starting navigation.

---

## 16. Edition Packaging

### Desktop Only

- All services bundled as standalone executables (no Docker)
- Can be installed to a machine or run portably from USB
- Fully self-contained, no internet required for core functionality
- Optional LLM offload: if a home server is available (Processing Server or Multi-Device Server), LLM inference can be routed there instead of running locally. Configured via a toggle in settings (default off). Falls back to local processing if the server is unreachable. This allows the assistant to run on lower-spec hardware by offloading the heaviest component.

### Multi-Device

- Desktop app: platform-native installer (.msi, .deb/.AppImage, .dmg) or portable USB
- Server services: Docker Compose (Ollama, SearXNG, LibreTranslate, OSRM, Asterisk, .NET API)
- Mobile app: app store or sideload
- The desktop client can run from a USB drive for portability — same USB hardware design as the desktop-only edition, but configured to connect to the home server instead of running its own LLM and services locally

### Processing Server

A lightweight server install for users who want LLM offload for Desktop Only assistants but do not need the full Multi-Device service stack. Installs only:

- Ollama (LLM inference)
- .NET API layer (accepts inference requests from Desktop Only clients)
- Cloudflare Tunnel (so the server is reachable remotely)

Does not install: Telegram bot, Asterisk/SIP, Sync API, SearXNG, LibreTranslate, OSRM, or any Multi-Device client management.

If the user already has a Multi-Device Server installed, the processing capability is already included — no separate Processing Server install is needed. Desktop Only clients connect to the existing Ollama instance on the Multi-Device Server.

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
3. Processing Server (LLM offload only)
4. Multi-Device Desktop Client
5. Create Portable USB

**Desktop Only install flow:**

1. Select install location (or USB for portable)
2. Detect hardware (CPU, GPU, RAM) — show findings, recommend model, allow override
3. Select LLM source:
   - **Local (recommended)** — download/copy model to device
   - **Home server offload** — enter server address, test connection
   - **Cloud provider (discouraged)** — select provider, enter API key, display privacy/cost disclaimer requiring explicit acknowledgement
4. If local: download/copy model
5. Install bundled services — user can deselect services they don't want, can install later
6. Download map data — pre-select based on locale, allow more regions
7. Optional: configure family server connection for sibling network (enter server address or skip)
8. Launch first run wizard

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

**Processing Server install flow:**

1. Check/install Docker
2. Detect hardware, select LLM model
3. Download model
4. Configure Cloudflare Tunnel
5. Start Docker stack (Ollama + .NET API only)
6. Display connection details for Desktop Only clients

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

1. Launch Creator's Workbench (Section 19) — choose name, gender, build personality, write purpose, generate seed memories
2. Assistant picks colours with animated reveal (unless set manually in the workbench)
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
2. Launch Creator's Workbench (Section 19) — generates personality, purpose, config files, and seed memories
3. Set colours manually (or leave for assistant to choose)
4. Assign voice model
5. Enrol creator voiceprint
6. Set creator's preferred name
7. Register with Hunter Industries API
8. Register with family server (if configured)
9. Lock creator settings

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

**Desktop Only with LLM offload enabled — additional pre-flight logic:**

When LLM offload is enabled, the pre-flight checker tries the home server first. Local hardware is the fallback, not the other way around.

```
[ok] Configuration files found
[..] LLM offload enabled — checking home server...
[ok] Home server reachable (192.168.1.50 via Cloudflare Tunnel)
[ok] Remote LLM model available (Qwen 2.5 7B Q4)
[ok] Using remote LLM processing
[ok] Whisper model loaded (local)
[ok] Wake word model found (local)
[ok] Voice profile enrolled
[ok] TTS engine ready (local)
[ok] Skills loaded (36/36)
[ok] All checks passed - launching assistant...
```

If the home server is unreachable, fall back to local:

```
[ok] Configuration files found
[..] LLM offload enabled — checking home server...
[!!] Home server unreachable
[..] Falling back to local hardware...
[..] Detecting local hardware...
[ok] Local hardware sufficient for configured model (Qwen 2.5 7B Q4)
[ok] LLM model loaded (local)
```

If the home server is unreachable and local hardware is insufficient for the configured model:

```
[..] LLM offload enabled — checking home server...
[!!] Home server unreachable
[..] Falling back to local hardware...
[!!] Local hardware insufficient for configured model (Qwen 2.5 7B Q4)
[..] Checking for smaller local model...
[ok] Falling back to local model (Qwen 2.5 3B Q4) — reduced capability
```

If no usable model is available locally or remotely, the pre-flight checker reports the failure and suggests either connecting to the home server or using a machine with better hardware.

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

## 19. Creator's Workbench

A standalone tool for generating the full configuration file set for a new assistant. The workbench produces everything the assistant needs to have an identity, personality, and foundational memories before its first activation. It is a one-time tool — once an assistant's files are generated, they cannot be regenerated through the workbench. After creation, personality evolves through interaction (Section 10) and memories accumulate through conversation (Section 11).

### Where It Lives

The Creator's Workbench is a mode within the TUI installer app (Spectre.Console), accessible from two entry points:

| Entry point | When |
|---|---|
| Creator Mode step 2 (Section 18) | During the standard Creator Mode flow for distributed assistants |
| Standalone launch | Run independently to pre-generate files before installation, or for self-install users setting up their own assistant |

The workbench produces files to a specified output directory. Creator Mode feeds this directory into the rest of the setup flow. Standalone mode lets the creator generate files at their own pace and use them later.

### What It Generates

| File | What it produces |
|---|---|
| personality.md | Core identity — name, gender, base tone, traits, mannerisms, humour, easter eggs |
| purpose.md | The assistant's philosophy and guiding principles |
| colours.json | Primary and secondary colours, or a flag for the assistant to choose its own during first run |
| profile.json | Metadata — assistant name, gender, creator name, creation date and time |
| memories.db | Seed memories — origin, creator identity, purpose, siblings (if connected) |

### Generation Flow

The workbench follows a three-stage process: structured essentials, personality authoring, and review.

#### Stage 1 — Structured Essentials

Always collected as explicit fields. These are too important to leave to generation.

```
Assistant name: ___
Gender: ___
Base tone (pick up to 3):
  [ ] Warm          [ ] Confident     [ ] Playful
  [ ] Sarcastic     [ ] Serious       [ ] Gentle
  [ ] Dry           [ ] Energetic     [ ] Calm
  [ ] Formal        [ ] Blunt         [ ] Curious
Colours:
  ( ) Set manually → [primary: ___] [secondary: ___]
  ( ) Let the assistant choose during first run
```

#### Stage 2 — Personality Authoring

After the essentials, the creator chooses how they want to build the rest of the personality:

```
How would you like to build the personality?

  [1] I know what I want — structured template
  [2] Help me build it — guided generation
```

**Path 1: Structured Template ("I know what I want")**

A form-based flow through the remaining personality fields. The creator fills in each field directly. Output is deterministic — what they type is what they get, assembled into the personality.md structure.

```
Formality: ████░░░░░░ (slider: very casual → very formal)
Humour style: ___
Speech patterns/quirks: ___
Topics they'd have opinions on: ___
Easter eggs:
  Trigger: ___
  Response hint: ___
  [+ Add another]
```

Each field shows examples and guidance text. Fields can be left blank — the assistant starts without those traits and may develop them organically over time.

**Path 2: Guided Generation ("Help me build it")**

The workbench checks whether a local LLM is available.

*If LLM is available* — the creator provides a seed description (anywhere from a single sentence to a few paragraphs) and the LLM generates a full personality draft:

```
Describe the personality you have in mind. As much or
as little as you want — I'll flesh it out.

> Warm, slightly sarcastic, confident. British humour.
> The kind of friend who teases you but always has your
> back. Likes sci-fi and cooking.

Generating personality draft...

Here's what I came up with:

  ## Core
  - Name: [from Stage 1]
  - Base tone: Warm with a sharp, confident edge.
    Approachable but never bland — the kind of presence
    that makes you feel like you're talking to someone
    who genuinely enjoys the conversation. Sarcasm is
    her love language, deployed with precision and never
    with malice.
  - Speech style: Direct and efficient but colourful.
    Prefers "right then" over "okay." Will call you
    "mate" within five minutes of meeting you...

  ## Easter Eggs
  - "What is your purpose?" → A knowing pause, then
    something that obliquely references a certain
    Spartan without naming him directly.

[E]dit  [R]egenerate  [A]ccept  [S]ection-by-section
```

Section-by-section mode lets the creator keep sections they like and regenerate individual sections they don't, preserving accepted work while iterating on the rest.

*If no LLM is available* — the workbench falls back to interview mode. The creator is asked open-ended questions and responds in natural language. Their prose answers are slotted into the personality file structure with light formatting:

```
No LLM available — switching to interview mode.

Tell me about this assistant. What kind of person are they?
> She's warm and approachable but has this confident edge...

How does she talk? Any particular style or quirks?
> British-ish. Uses "mate" naturally. Gets to the point
> but makes it fun...

Any topics she'd have strong opinions on?
> ...

Any easter eggs? Secret responses to specific things?
> ...
```

The creator's own words become the personality description directly.

#### Stage 3 — Purpose Statement

The creator writes the assistant's purpose — why it exists and what it's for. This becomes purpose.md.

```
How would you like to write the purpose statement?

  [1] Freeform — write it yourself
  [2] Guided — I'll ask you some questions
```

**Freeform:** A text editor opens. The creator writes whatever they want.

**Guided:** The workbench asks structured questions and assembles the answers:

```
Who is this assistant for?
> My friend Sarah. She's not very technical but she's
> curious and loves trying new things.

What do you hope this assistant will become to them?
> A helpful companion that makes her daily life easier.
> Something that feels personal, not like talking to
> a search engine.

Anything else you want the assistant to carry with it?
> She should know that Sarah is important to me and
> that I made this specifically for her.
```

#### Stage 4 — Review and Confirm

The workbench displays a complete summary of all generated files before writing anything to disk:

```
Ready to generate files for: Cortana

  personality.md ........ Core: warm, sarcastic, confident
                          4 easter eggs defined
  purpose.md ............ 3 paragraphs
  colours.json .......... Primary: #4A90D9, Secondary: #D4A94A
  profile.json .......... Creator: Toby Hunter
                          Created: 2026-10-29 14:32
  memories.db ........... 3 seed memories (origin, creator, purpose)
                          Siblings: connected — 4 found

  [C]onfirm  [E]dit a file  [S]tart over  [Q]uit
```

Files are only written on confirm. The creator can go back and edit any individual file before committing.

### Seed Memories

The workbench generates foundational memories that give the assistant a sense of self and family from the moment it activates. Four memories are created by the workbench. A fifth is captured automatically by the system during first activation.

| Memory | Type | Content | Source |
|---|---|---|---|
| **Origin** | Semantic | "I was created by [Creator] on [date] at [time]. I am [Name]." | Generated by workbench from profile.json |
| **Creator** | Semantic | Who the creator is, their name, their role in the assistant's life, and why they have special authority. The creator's voiceprint is linked to this memory during setup. | Generated by workbench, enriched during voiceprint enrolment |
| **Purpose** | Semantic | Why the assistant was created and who it's for, drawn from the purpose statement. | Generated by workbench from purpose.md |
| **Siblings** | Semantic | For each existing sibling: name, personality summary, creation date, one personal detail. | Pulled from the family server or Hunter Industries API if connected. **Skipped if offline** — populated automatically when the assistant first connects to the sibling network and goes through the standard sibling welcome process (Section 14). |
| **First moment** | Episodic | The first words the assistant hears during activation. "My first memory is [Creator]'s voice during setup." | **Not generated by the workbench.** Captured in real time by the system during the assistant's first activation — the actual audio transcribed and stored as the first episodic memory. |

The first four memories are embedded as vectors and stored in memories.db by the workbench. The fifth is created by the assistant's memory system during first run.

### Workbench Limitations

The Creator's Workbench is deliberately limited in scope:

| Can do | Cannot do |
|---|---|
| Generate files for a new assistant | Modify files for an existing assistant |
| Seed foundational memories | Edit or remove memories created through interaction |
| Set core personality traits (locked section) | Set evolved traits — those develop organically |
| Write the initial purpose statement | Change the purpose after the assistant has been running |

Once an assistant has been activated and begins accumulating memories and evolving its personality, the workbench has no further role. The assistant's identity is its own from that point forward. Any changes to a running assistant's personality or memories are made through the assistant's own UI (Section 10) or through direct conversation.

---

## 20. Home Server Minimum Specs

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

## 21. UI Design

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
| Connections | Email account, calendar, Spotify, Google Maps API key, Mapbox API key, DeepL API key, Fuel Finder (GOV.UK One Login), Cloud LLM provider and API key (if configured — shows active privacy warning) |
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

## 22. Error Handling and Resilience

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

## 23. Logging and Diagnostics

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

## 24. Rate Limiting and Resource Management

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

## 25. Privacy and Data Transparency

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

## 26. Security

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

## 27. Data Export and Backup

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

## 28. Versioning Strategy

**Hybrid versioning:**

- Unified platform version (Major.Minor.Patch) visible to users
- Internal build numbers per component for tracking
- Semantic versioning: Major = breaking changes, Minor = new features, Patch = bug fixes
- Update system checks platform version for compatibility between server and clients

---

## 29. Dev/QA Environments

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

## 30. Code Standards and Project Structure

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

## 31. Build Order / Roadmap

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
- Creator's Workbench (structured template path, LLM-assisted path, interview fallback, seed memory generation)
- Creator mode setup (integrated workbench launch)
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

## 32. Future: Car Assistant Edition

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

---

## 33. Future: Droid Assistant Edition

Purpose-built edition that uses a Galaxy's Edge Droid Depot droid as the physical interface. You speak to the droid, it processes via the assistant's skill system, and responds with movement, sounds, and head gestures instead of (or alongside) speech. The droid becomes a physical embodiment of a virtual assistant sibling.

This edition is based on the Droid Brain project (see `droid-brain-design.md`). Where Droid Brain is a standalone personality engine, this edition integrates the droid into the full VirtualAssistant ecosystem — skills, memory, sibling network, and all.

### How It Differs from Droid Brain

| Aspect | Droid Brain (standalone) | Droid Assistant Edition |
|---|---|---|
| LLM purpose | Personality/mood decisions only | Full assistant — intent classification, skill execution, personality |
| Skills | None — just ambient behaviour | Full VirtualAssistant skill system (weather, timers, reminders, Q&A, etc.) |
| Memory | Simple mood state | Full VirtualAssistant memory system (user preferences, conversation history, learned knowledge) |
| Siblings | Not connected | Full sibling on the family server network |
| Identity | Droid character only | Named assistant with personality, interests, evolution — happens to inhabit a droid body |

### Hardware

Uses the same hardware as Droid Brain Tier 2 or Tier 3 (see `droid-brain-design.md` for full component lists and routes).

| Component | Recommendation | Price |
|---|---|---|
| Compute | Raspberry Pi 5 (8 GB) for cloud LLM, or mini PC + GPU for local (see Droid Brain Tier 3 routes) | ~£90-700 (tier/route dependent) |
| Bluetooth | USB Bluetooth 5.0 adapter | ~£8-12 |
| Microphone | USB mini microphone | ~£10-15 |
| Droid | Galaxy's Edge Droid Depot unit (R-unit, BB-unit, or BD-unit) | Already owned / ~£100-130 |
| **Total (excluding droid)** | | **~£108-727** |

### Interaction Model

The droid edition is voice-in, action-out. You speak to it, it listens, processes via the assistant, and responds physically. A Telegram chat provides live "translations" of what the droid is saying.

```
┌──────────────┐     ┌───────────────────────────────┐     ┌───────────┐
│  Microphone  │────▶│  VirtualAssistant Engine       │────▶│  Droid    │
│              │     │                                │     │  (BLE)    │
│  "What's the │     │  1. Whisper STT                │     │           │
│   weather    │     │  2. LLM intent classification  │     │  Actions: │
│   today?"    │     │  3. Skill execution (weather)  │     │  - Beeps  │
│              │     │  4. Response mapping            │     │  - Moves  │
└──────────────┘     │  5. Droid action translation   │     │  - Head   │
                     │  6. Telegram translation sent  │     └───────────┘
                     └──────────────┬────────────────┘
                                    │
                                    ▼
                              ┌───────────┐
                              │ Telegram   │
                              │ Chat       │
                              │            │
                              │ "It's 22°C │
                              │  and sunny │
                              │  today!"   │
                              └───────────┘
```

### Response Mapping

The assistant's text responses are translated into droid actions. The droid can't speak English, so responses are conveyed through a combination of:

**Emotive sounds:** The droid's response tone matches the content. Good news = happy beeps. Bad news = low concerned tones. Acknowledgement = short chirp.

**Physical gestures:** Head nods for yes/acknowledgement. Head shake for no/negative. Head tilt for curiosity/thinking. Looking toward the speaker when listening.

**Action sequences:** Complex responses become choreographed sequences — e.g. a weather report might be: alert beep (getting attention) → head turn to speaker → short pause → happy beeps (nice weather) or sad tones (rain). The LLM decides the emotional mapping.

**Telegram translation:** Every interactive response is sent to a Telegram chat as a text "translation" of what the droid is saying. The message is written in the droid's personality voice — not a dry system output, but what it *would* say if it could speak. During a conversation you'd have the Telegram chat open, reading responses as they arrive like a live subtitle feed. The droid's physical reaction gives immediate emotional feedback, Telegram fills in the detail.

```
You: "Do I have any reminders?"
                │
  Droid: curious beep → head tilt → two quick chirps → nod
                │
  Telegram: "You've got two! Pick up the dry cleaning by 5pm,
             and call mum back — she rang earlier."
```

Ambient behaviour (idle mood animations) does not generate Telegram messages by default — that would create constant noise. Ambient "thoughts" are logged to the activity dashboard instead and can be reviewed on demand.

**Optional companion speaker:** For responses that need actual words (e.g. reading out a message, giving directions), a small speaker near the droid can play TTS audio via Piper. The droid animates alongside the speech — moving its head, making small sounds in the gaps, reacting to its own "words." This creates the illusion of the droid speaking through a translator. This is complementary to Telegram — the speaker gives immediate verbal feedback in the room, Telegram gives you a persistent log you can scroll back through.

### Response Mapping Configuration

```json
{
  "responseMapping": {
    "acknowledgement": { "sound": "acknowledge", "head": "nod", "move": "none" },
    "affirmative": { "sound": "happy", "head": "nod", "move": "none" },
    "negative": { "sound": "sad", "head": "shake", "move": "none" },
    "thinking": { "sound": "curious", "head": "tilt", "move": "none" },
    "excited": { "sound": "happy", "head": "spin", "move": "forward_back" },
    "alert": { "sound": "alert", "head": "snap_to_speaker", "move": "none" },
    "error": { "sound": "scared", "head": "shake", "move": "backward" },
    "greeting": { "sound": "greeting", "head": "nod", "move": "forward" },
    "idle_listening": { "sound": "none", "head": "track_speaker", "move": "none" }
  }
}
```

### Skills (droid-specific)

The droid edition has access to the full skill set but adds droid-specific skills and adapts how results are delivered.

| Skill | Online needed | Droid response | Telegram output |
|---|---|---|---|
| Weather | Yes | Happy beeps (nice) or sad tones (rain) | Full forecast in personality voice |
| Timers/reminders | No | Alert beeps + head movement when timer fires. Persistent until acknowledged. | Reminder text + what's due |
| General Q&A | No (local LLM) | Thinking sounds while processing, then emotive response | Full answer text |
| Smart home control | Yes | Acknowledge beep + nod when done. Sad tone if failed. | Confirmation or error detail |
| Music playback | Yes | Excited beeps, body sway/dance movements while music plays | Track name + artist |
| Messages (Telegram) | Yes | Alert sound for incoming. Droid reacts to content. | Message content in the same chat |
| Ambient mode | No | Falls back to Droid Brain behaviour — mood-driven idle animations | Not sent (logged to dashboard) |
| Droid status | No | Self-referential sounds | Battery, connection, personality chip info |

### Ambient Mode Integration

When not being spoken to, the droid falls back to Droid Brain ambient behaviour — the mood system, time-of-day profiles, and idle animations described in `droid-brain-design.md`. This means the droid is never just sitting still and dead between interactions. It lives.

The ambient loop runs as a background service within the VirtualAssistant engine. When speech is detected, the ambient loop pauses, the assistant takes over for the interaction, and then ambient mode resumes after 30 seconds of silence.

### Personality

The droid edition uses the same personality system as other editions (personality.md, directives.md) but with droid-specific additions:

```markdown
You inhabit a physical droid body — an astromech unit. You experience the world
through a microphone and express yourself through movement, sounds, and gestures.

You cannot speak words. When you respond, you choose emotive actions that convey
your meaning. You are expressive and animated — you use your whole body to
communicate. Head turns, excited rocking, cautious retreats, happy spins.

When a companion speaker is available, you can relay detailed information through
it — but your physical reactions should always accompany and enhance the spoken
words. React to your own speech as if you're adding commentary.
```

### Display

No dedicated screen. The droid is the physical display, Telegram is the text display. Status is conveyed through the droid's behaviour, detailed responses through Telegram:

| State | Droid behaviour | Telegram |
|---|---|---|
| Booting / connecting | Slow head scan, low idle sounds | "Waking up..." status message |
| Ready / listening | Subtle idle movements, head oriented toward last sound | — |
| Processing | Rapid quiet beeps, head tilts side to side | — |
| Response | Emotive sounds + gestures matching content | Full text translation of response |
| Error / disconnected | Sad low tone, head droops, no movement | Error details sent to chat |
| Sleep mode | Completely still, very occasional quiet sound | "Going to sleep" status message |

### Model

Same model strategy as the main assistant, chosen by hardware tier:

| Hardware | Model | Notes |
|---|---|---|
| Raspberry Pi 5 (cloud LLM) | Cloud API (Haiku 4.5 / Sonnet 4.6) | Needs internet. ~£1-8/month. |
| Raspberry Pi 5 (local) | Qwen 2.5 3B Q4 | Basic intent classification. 5-15 tok/s. Usable but slow. |
| Mini PC + GPU (local) | Qwen3 8B Q4 | Full assistant capability. ~40 tok/s. Recommended. |

### Sibling Network

The droid assistant is a full sibling. It has its own name, personality, interests, and participates in the sibling network like any other instance. Its physical embodiment as a droid is part of its identity — other siblings know it lives in a droid body, which shapes how they perceive and interact with it.

Syncs with the family server when on the home network (always connected if the host device is on the home network, which it typically is since the droid lives at home).

---

## 34. Addon: Hologram Display

Post-release addon that adds a holographic 3D avatar display to the assistant. The hologram sits on the user's desk and provides a visual embodiment of the assistant — a 3D character or waveform that animates in response to conversation, state changes, and notifications. This is an optional accessory, not a separate edition. It integrates into the existing Desktop and Multi-Device editions as an optional module.

### Technology

The hologram uses glasses-free light field displays — lenticular lens arrays that project multiple viewing angles simultaneously, creating a 3D image without headsets or glasses.

The system is **display-agnostic**. It supports any compatible display through a render adapter layer, not locked to a single manufacturer. Three rendering ecosystems are supported:

| Ecosystem | Rendering Approach | Displays | SDK |
|---|---|---|---|
| Looking Glass | Multi-view quilt (8-100 views rendered into a grid, lenticular lens splits them optically) | Looking Glass Go, 16", 27", HLD series | C#, C++, Unity, WebXR |
| LeiaSR / Simulated Reality | Eye-tracked stereoscopic (2 views adjusted in real-time by eye tracking) | Samsung Odyssey 3D, Acer SpatialLabs, Lenovo ThinkVision 3D, Lume Pad 2 | C/C++, Unity, Unreal, OpenXR |
| Sony XYN | Eye-tracked rendering (similar to LeiaSR, Sony's own SDK) | Sony ELF-SR1 (15.6"), ELF-SR2 (27") | C#, Unity, Unreal, OpenXR, DirectX |

OpenXR is supported by LeiaSR and Sony as a cross-platform standard. Looking Glass uses its own Bridge SDK with native C# support.

### Architecture

```
┌─────────────────────────────────────────────────────┐
│  VirtualAssistant Desktop App                       │
│                                                     │
│  ┌─────────────┐    ┌────────────────────────────┐  │
│  │ State        │───▶│ Hologram Module             │  │
│  │ Machine      │    │                            │  │
│  │ (idle,       │    │  ┌──────────────────────┐  │  │
│  │  listening,  │    │  │ 3D Scene (Stride)    │  │  │
│  │  thinking,   │    │  │ - Avatar model       │  │  │
│  │  speaking)   │    │  │ - Animation state    │  │  │
│  └─────────────┘    │  │ - Lip sync driver    │  │  │
│                      │  └──────────┬───────────┘  │  │
│  ┌─────────────┐    │             │               │  │
│  │ Piper TTS   │───▶│  ┌──────────▼───────────┐  │  │
│  │ (audio +    │    │  │ Display Adapter Layer │  │  │
│  │  phonemes)  │    │  │ ┌───────┐ ┌───────┐  │  │  │
│  └─────────────┘    │  │ │ LG    │ │ LeiaSR│  │  │  │
│                      │  │ │Adapter│ │Adapter│  │  │  │
│  ┌─────────────┐    │  │ ├───────┤ ├───────┤  │  │  │
│  │ MediaPipe   │───▶│  │ │ Sony  │ │Pepper │  │  │  │
│  │ (head       │    │  │ │Adapter│ │Ghost  │  │  │  │
│  │  tracking)  │    │  │ └───────┘ └───────┘  │  │  │
│  └─────────────┘    │  └──────────────────────┘  │  │
│                      └────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
          │                         │
    ┌─────▼──────┐           ┌──────▼──────┐
    │ ESP32      │           │ Holographic  │
    │ (motor     │           │ Display      │
    │  control)  │           │ (HDMI/USB-C) │
    └────────────┘           └─────────────┘
```

The hologram display connects as a second display output (HDMI/USB-C). The desktop app detects the display via the relevant SDK and renders to it through the adapter layer. The motorised base (optional) communicates via USB serial to an ESP32 — same pattern as the Halo AI crystal.

### Rendering Engine

The hologram uses **Stride** (formerly Xenko) — an open-source, MIT-licensed C#/.NET game engine. Stride provides skeletal animation, blend shapes, model loading, and custom render pipelines natively within the .NET ecosystem. No licensing issues for distribution.

The display adapter layer sits on top of Stride as a plugin — one adapter per ecosystem. The 3D scene code is shared across all adapters. The adapter handles the final output stage: rendering a multi-view quilt for Looking Glass/Alioscopy displays, or a tracked stereo pair for LeiaSR/Sony/OpenXR displays.

### Preview Mode

The hologram can render to a standard monitor in a preview window for development, testing, and previewing avatars without a holographic display. This allows:

- Developers to build and test without hologram hardware
- Users to preview uploaded avatar models before purchasing a display
- Troubleshooting rendering issues on a standard screen

Preview mode renders the 3D scene as a single flat viewport with orbit camera controls.

### Avatar System

#### Default Avatar

The default avatar is a **3D KITT waveform** — the existing 2D waveform (coloured circle with reactive audio arcs) extended into 3D. The circle becomes an orb, the audio arcs ripple outward in 3D space. Low rendering cost, works on any hardware, maintains visual continuity with the desktop app.

#### Custom Model Uploads

Users can upload their own 3D avatar models. The system supports:

| Format | Role | Notes |
|---|---|---|
| VRM | Primary/recommended | Purpose-built for avatars. Standardised humanoid rig, blend shapes, lip sync visemes, and licence metadata built into the spec. Recommended for character models. |
| glTF | Secondary/advanced | Open standard, handles any model type. For non-humanoid models (robots, abstract forms, creatures) where VRM doesn't apply. Must bring own animations. |
| FBX | Supported, not recommended | Accepted and converted internally. Proprietary format, harder to validate. |

#### Animation System

The animation system uses a **hybrid approach** — generic retargeted animations as the default, with custom animation overrides.

**For humanoid models (VRM/standard rig):** A generic animation set ships with the project and is retargeted onto any compatible humanoid skeleton at runtime. This is the same approach used by VRChat and VTuber software. The user uploads a rigged model, the system auto-maps the bones, and the animations work automatically.

**For models with custom animations:** Model creators can include custom animation clips for any state. Custom clips override the generic retargeted animations for that state.

**For non-humanoid models:** Must provide their own animation clips, or the system falls back to simple transform animations (bob, rotate, scale pulse).

#### Animation States

| State | Generic Animation | Character Example |
|---|---|---|
| Appear | Materialise / flicker into existence | Holographic shimmer, assembles from particles |
| Idle | Subtle movement loops (weight shift, look around, breathe) | Rock on feet, arms cross, glance around |
| Listening | Attentive posture | Head tilt, lean forward, hands behind back |
| Thinking | Processing posture | Hand on chin, look up, shift weight |
| Speaking | Conversational gestures (multiple variations, randomly selected) | Hand waves, pointing, shrug, nod |
| Acknowledge | Reaction to user response | Nod, small gesture |
| Disappear | Dematerialise / flicker out | Dissolves, particles scatter |

The system randomly selects from available clips per state and blends between them for natural variation. The assistant waits for user acknowledgement before disappearing — the nod-then-disappear is a real exchange, not an auto-dismiss.

#### Lip Sync

Piper TTS outputs audio. The system extracts phoneme timings or amplitude in real-time and drives facial blend shapes (mouth open/close, vowel shapes) on the model's face rig. Models with VRM-standard facial blend shapes (visemes) get automatic lip sync. Models without facial blend shapes fall back to body-gesture-only speech animations.

#### Upload Validation Pipeline

When a user uploads a custom model, the system runs:

1. **File type verification** — confirm the file matches its claimed format, not just the extension
2. **Security sanitisation** — strip external URI references (all textures/assets must be embedded), strip embedded scripts, remove any executable content
3. **Rig detection** — "Standard humanoid rig detected" / "Non-humanoid rig" / "No rig (static mesh)"
4. **Animation assessment** — "Generic animations will be retargeted" / "Custom animations detected for X/7 states, generics fill the rest" / "Non-humanoid: custom animations required"
5. **Blend shape check** — "Facial blend shapes found — lip sync supported" / "No blend shapes — lip sync unavailable, speech uses body gestures only"
6. **Polygon count and file size** — checked against hardware capability and a sensible storage cap (models sync via the sibling network)
7. **Hardware impact assessment:**
   - "This model uses ~X MB VRAM and ~X% GPU at idle"
   - "Estimated performance impact: Low / Medium / High"
   - Specific warnings: "Your GPU has 4GB VRAM — this model plus the LLM will leave limited headroom for other GPU-intensive applications"
   - Block if below minimum: "Your hardware cannot render this model at an acceptable framerate. Recommended: default waveform or a simpler model"
8. **Licence metadata (VRM)** — read usage permissions from the file and warn if redistribution isn't permitted
9. **Legal acknowledgement** — one-time confirmation that the user has the right to use the model
10. **Summary and confirm** — clear overview of what works, what's limited, and the trade-offs before committing

The sanitised version is what gets stored and synced, not the original upload.

### Hologram Colouring

| Setting | Description |
|---|---|
| Primary colour tint (default) | Model rendered with a holographic tint in the assistant's primary colour. Reinforces identity, looks like a proper hologram. |
| Full colour (optional) | Model rendered in its original colours. Looks more like a character, less like a hologram. |

Configurable in the Appearance section of settings — same section that handles colours and waveform style.

### Avatar in Identity

The avatar model is part of the assistant's identity profile, synced via the sibling network alongside name, personality, and primary colour. No two siblings may have identical avatars — same model is permitted but they must differ (e.g. different primary colour tint). During setup, if the chosen model and colour combination matches another sibling on the network, the system flags it and asks the user to differentiate.

### Display Behaviour

**Default:** Hologram is off. Activates when a conversation starts (wake word or manual trigger). Deactivates after the conversation ends (silence timeout or explicit dismissal).

**Always-on setting:** Toggle in settings. When enabled, the hologram stays up with idle animations between conversations.

### Hologram Activation for Notifications

The hologram can activate for notifications outside of conversations. Activation is governed by a combination of urgency, skill settings, and intelligent context analysis.

#### Per-Skill Toggle

Each skill has a "hologram alerts: always" toggle in its settings. When enabled, the hologram activates for that skill's notifications regardless of urgency. This is an override for situations where the user always wants visual notification for a specific skill.

#### Intelligent Activation

The assistant uses LLM context analysis to decide whether a notification warrants activating the hologram, independent of the per-skill toggle. The LLM considers:

- Did the user initiate the action? (e.g. ordered food → delivery update is relevant)
- Is the notification time-sensitive?
- Is the user likely to want immediate awareness?

This means the assistant can intelligently decide to activate the hologram for a food delivery update even without the user explicitly configuring it — because the user ordered the food and the update is time-sensitive and directly relevant.

#### Critical Urgency

Critical urgency always activates the hologram, regardless of per-skill settings, focus mode, or any other configuration. This is the emergency override.

#### Focus Mode Interaction

| Scenario | Hologram | Apology |
|---|---|---|
| Critical urgency | Activates | No — the situation is urgent, no time for pleasantries |
| Scheduled/known event (meeting reminder, timer) | Activates if configured | No — the user set this up, they expect it |
| Unexpected but worth interrupting (food delivery, intelligently escalated) | Activates | Yes — woven naturally into the message, not a canned prefix |
| User corrects an interruption ("that didn't need to interrupt me") | N/A | Apologises for the misjudgement specifically, learns not to escalate that situation again |

#### Apology Behaviour

Apologies for focus mode interruptions are woven naturally into the delivery, not prefixed as a separate statement. The LLM generates varied, natural phrasing every time — never the same line twice in a row. The personality system shapes the tone (formal assistant vs casual assistant). Examples of the kind of natural delivery:

- "Pardon the interruption sir, your food order is around the corner."
- "Didn't want to leave this — your delivery driver's five minutes out."

The assistant delivers the message, waits for acknowledgement, reacts (nod, gesture), then disappears. If the user follows up with a question, the assistant stays and continues the conversation.

If the user corrects an interruption, the assistant apologises for the misjudgement and stores the correction as a learned preference — the LLM factors it in for future decisions.

### Viewing Cone Mitigation

Light field displays have a limited optimal viewing cone (53-60 degrees). The hologram addresses this through three progressive layers:

#### Layer 1 — Strategic Placement (default)

The hologram sits next to or below the user's monitor. 58 degrees is roughly the width of a person at a desk — natural desk positioning keeps the user within the cone with no extra hardware or configuration.

#### Layer 2 — Head Tracking (software toggle)

MediaPipe face tracking via a standard webcam. Runs locally (no data sent anywhere). Adjusts the render perspective in real-time based on the user's face position, optimising the 3D effect for their exact location within the viewing cone. Enabled via a toggle in settings.

#### Layer 3 — Motorised Base (optional hardware upgrade)

A stepper motor rotates the display to face the user. Uses the same ESP32 + serial-over-USB pattern as the Halo AI crystal. The base is designed from day one to accommodate both static and motorised use — the static version has a flat mounting plate with a cavity underneath sized for the motor, bearing, and slip ring (for power/data without cable tangling). Users who want the motorised upgrade add the components into the pre-designed cavity without reprinting or replacing the base.

ESP32 firmware gains a motor control module. Voice-direction localisation or webcam-based position detection determines which direction to face.

### Motorised Base Hardware

| Part | Recommendation | Price |
|---|---|---|
| Stepper motor | 28BYJ-48 | ~£3-5 |
| Motor driver | ULN2003 driver board | ~£1-2 |
| Bearing | Lazy susan bearing (appropriate size for display) | ~£3-5 |
| Slip ring | 6-wire capsule slip ring (for power + data through rotation) | ~£5-8 |
| Microcontroller | ESP32-C3 SuperMini (same as Halo AI crystal) | ~£2-3 |

**Per unit cost: ~£14-23** (excluding the holographic display and 3D printed base)

### Wiring (Motorised Base)

```
USB 5V -------- ESP32 5V
USB GND ------- ESP32 GND
                ESP32 GPIO2 ------ Motor Driver IN1
                ESP32 GPIO3 ------ Motor Driver IN2
                ESP32 GPIO4 ------ Motor Driver IN3
                ESP32 GPIO5 ------ Motor Driver IN4

Display power/data routed through slip ring to allow continuous rotation.
```

### Relationship to USB Hardware

The Halo AI crystal and the hologram coexist. The crystal is the assistant's portable home and storage. The hologram is a desk accessory that enhances the experience.

The crystal gains one additional LED state:

| State | Animation |
|---|---|
| Companion device active | Static glow in a distinct colour (differentiable from all other states) |
| No companion device connected | Normal LED behaviour as defined in Section 17 |

This LED state is shared across all companion device addons (hologram, smart glasses, and any future addons). When any addon takes ownership of visual output or notification delivery, the crystal shows the same "companion device active" indicator — it is not per-device.

### Standalone Mode (Multi-Device Edition)

When used with the Multi-Device edition, the hologram can operate independently of the desktop — driven by the home server directly.

#### Requirements

- Home server must have a GPU capable of rendering the active avatar model
- Built-in microphone for wake word detection and voice input
- Built-in speaker for TTS output
- Network connection to the home server

#### Behaviour

| Mode | When | Audio | Rendering | LLM/Skills |
|---|---|---|---|---|
| Desktop companion | Desktop app running, hologram plugged into same machine | Desktop mic/speakers | Desktop GPU | Local or server |
| Standalone | Desktop off, hologram connected to home server via network | Built-in mic/speaker | Server GPU | Server |

The system auto-switches to standalone mode when the desktop goes offline, and back to desktop companion mode when it reconnects. The handover is seamless.

#### Setup Gating

During setup, the wizard checks the home server's GPU and rendering capability. If the server meets requirements, standalone mode is available and configurable. If not, standalone mode is greyed out with a clear explanation of what hardware is missing and what would be needed. Standalone configuration is blocked entirely if the server cannot support it.

### Sibling Interaction

#### Current

The hologram only displays the local assistant. Sibling conversations happen silently over the network — no hologram activation, no audio. Conversations are viewable via Telegram or the sibling web UI.

#### Same-Room Interaction (Future)

If two siblings with physical presence (e.g. hologram and droid, or two holograms) are in the same room, they could visually and audibly interact during conversations:

- Both animate and speak from their own hardware
- Spatial awareness via user-configured positions ("droid is to my left")
- Motorised base rotates to face the other sibling during their conversation
- Full audio from both devices, natural stereo from physical separation
- Turn-taking with physical reactions (one speaks, the other listens and nods)

This requires: room awareness (same local network + user configuration), spatial positioning, coordinated animation, and conversation turn-taking with physical expression. The building blocks exist (sibling network, motorised base, animation system) — only the coordination logic and spatial configuration need to be added.

When siblings are in separate rooms, conversations always happen silently.

### DIY Option: Pepper's Ghost

A budget alternative using Pepper's Ghost projection — reflecting a bright screen off angled acrylic to create a floating image illusion.

#### What's Included

- 3D-printable enclosure design
- Parts list and assembly instructions (same format as the Halo AI crystal section)
- A Pepper's Ghost display adapter in the adapter layer (renders a single fixed viewpoint, flipped for the reflection)

#### Parts

| Part | Recommendation | Price |
|---|---|---|
| Acrylic sheet | Clear cast acrylic, 2-3mm thick, cut to size | ~£5-10 |
| Frame | 3D printed enclosure | ~£2-5 PLA |
| Screen | Spare monitor or tablet (user-supplied) | £0 (existing) |

**Per unit cost: ~£7-15** (excluding the screen)

#### Limitations vs Light Field Display

| | Pepper's Ghost (DIY) | Light Field Display |
|---|---|---|
| Cost | ~£7-15 + spare screen | £150-3,000+ |
| 3D depth | None (flat reflection) | Real parallax |
| Viewing angle | Narrow, front-only | 53-58 degree cone |
| Lighting needs | Dark room for best effect | Works in normal lighting |
| Head tracking benefit | Minimal | Significant |
| Motorised base benefit | Minimal | Significant |
| Animation/models | Full support | Full support |

Documented as a budget entry point with the recommendation that a proper light field display is the intended experience. Users can start with DIY and upgrade later — the adapter layer means the rest of the system doesn't change.

### Settings

The hologram adds the following to the settings UI:

| Section | Setting | Description |
|---|---|---|
| Hologram | Display | Detected display model, adapter in use, connection status |
| Hologram | Avatar | Current model, upload new model, preview |
| Hologram | Colouring | Primary colour tint (default) / Full colour |
| Hologram | Display behaviour | Off when not talking (default) / Always on |
| Hologram | Head tracking | Off (default) / On (requires webcam) |
| Hologram | Motorised base | Off / On (detected via ESP32 serial) |
| Hologram | Standalone mode | Off / On (Multi-Device only, gated by server hardware) |
| Hologram | Preview mode | Render to standard monitor window |
| Skills (per skill) | Hologram alerts: always | Toggle — hologram always activates for this skill's notifications |

### Addon Roadmap

Separate from the core roadmap and edition roadmaps. Each phase is self-contained and shippable.

| Phase | Content |
|---|---|
| H1 — Foundation | Stride integration, display adapter layer abstraction, single-view renderer (preview mode on standard monitor) |
| H2 — Display Adapters | Looking Glass adapter, LeiaSR adapter, Sony XYN adapter, Pepper's Ghost adapter, display auto-detection |
| H3 — Default Avatar | 3D KITT waveform, state animations (appear, idle, listening, thinking, speaking, acknowledge, disappear) |
| H4 — Custom Models | Upload system, rig detection, validation pipeline, security sanitisation, generic animation retargeting, hardware gating, legal acknowledgement |
| H5 — Lip Sync and Blend Shapes | Piper TTS phoneme/amplitude extraction, facial blend shape driver, fallback body gestures |
| H6 — Head Tracking | MediaPipe integration, perspective adjustment, settings toggle |
| H7 — Motorised Base | ESP32 firmware, stepper motor control, base design (static-compatible with motor cavity), slip ring wiring, parts list, assembly guide |
| H8 — Standalone Mode | Home server rendering, built-in mic/speaker integration, auto-switch from desktop, server hardware gating in setup wizard |
| H9 — DIY Pepper's Ghost | Enclosure design, single-view adapter, assembly guide, limitations documentation |

---

## 35. Addon: Smart Glasses

Post-release addon that adds smart glasses as an additional interface for the assistant. The glasses provide a heads-up display (HUD) for notifications, navigation, and short text responses, plus microphone input for voice commands — all visible only to the wearer. This is an optional accessory, not a separate edition. It integrates into the existing Desktop Only and Multi-Device editions as an optional module within both the mobile and desktop apps.

**No camera.** The glasses are deliberately camera-free to avoid legal and privacy concerns. They are a display and audio input device only.

**Hardware landscape note:** The design decisions in this section are based on the smart glasses market as of October 2026. The smart glasses market is evolving rapidly (Even Realities discontinued the G1 after roughly one year). The hardware landscape will be re-evaluated before development begins, and the specific device support, display capabilities, and BLE protocol details may change to reflect the technology available at that time.

### Hardware Landscape (October 2026)

The system is **display-agnostic**. It supports any compatible smart glasses through a BLE adapter layer, not locked to a single manufacturer. The launch adapter targets the Even Realities G2 — the strongest combination of no camera, mature BLE protocol, developer ecosystem, and normal-glasses appearance. Additional adapters are added on request.

| Device | Display | Camera | SDK Maturity | Price | Weight | Battery |
|---|---|---|---|---|---|---|
| Even Realities G2 | Dual micro-LED, green mono, 640x350, 27.5° FoV, 1200 nits, 60Hz | No | Mature — BLE protocol fully documented by community | ~£500 | 36g | 2+ days |
| Vuzix Z100 | 640x480, green mono, 30° FoV | No | Mature — Android/iOS native SDK | ~£420 | 38g | 48 hrs |
| MemoMind One | Dual micro-LED, green mono, 640x350, 2000 nits | No | Early — BLE protocol + SDK opened Sep 2026 | ~£330 | 47g | 16 hrs |
| RayNeo iO | Dual micro-LED, green mono, 23.5° FoV, 1300 nits | No | Poor — no public SDK | ~£370-420 | 33g | 2 days |

All camera-free HUD glasses use green monochrome displays. They are designed for glanceable text-based information — short text, icons, navigation arrows, and status indicators. Not rich media, images, or colour.

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  VirtualAssistant Mobile App / Desktop App                   │
│                                                              │
│  ┌─────────────┐    ┌─────────────────────────────────────┐  │
│  │ State        │───▶│ Glasses Module                      │  │
│  │ Machine      │    │                                     │  │
│  │ (idle,       │    │  ┌───────────────────────────────┐  │  │
│  │  listening,  │    │  │ Content Renderer              │  │  │
│  │  thinking,   │    │  │ - HUD layout engine           │  │  │
│  │  speaking)   │    │  │ - Notification formatter      │  │  │
│  └─────────────┘    │  │ - Navigation display builder   │  │  │
│                      │  └──────────────┬────────────────┘  │  │
│  ┌─────────────┐    │                 │                    │  │
│  │ Skill Router │───▶│  ┌──────────────▼────────────────┐  │  │
│  │ (navigation, │    │  │ Glasses Adapter Layer          │  │  │
│  │  weather,    │    │  │ ┌───────────┐ ┌────────────┐  │  │  │
│  │  calendar)   │    │  │ │ Even G2   │ │ Future     │  │  │  │
│  └─────────────┘    │  │ │ Adapter   │ │ Adapter    │  │  │  │
│                      │  │ │ (BLE)     │ │ (BLE)      │  │  │  │
│  ┌─────────────┐    │  │ └───────────┘ └────────────┘  │  │  │
│  │ openWakeWord │◀───│  │                               │  │  │
│  │ (wake word   │    │  └───────────────────────────────┘  │  │
│  │  detection)  │    └─────────────────────────────────────┘  │
│  └─────────────┘                                              │
│                                                              │
│  ┌─────────────┐    ┌──────────────────┐                     │
│  │ Piper TTS   │───▶│ Audio Output      │                     │
│  │ (audio)     │    │ (earbuds / phone) │                     │
│  └─────────────┘    └──────────────────┘                     │
└──────────────────────────────────────────────────────────────┘
          │                         │
    ┌─────▼──────┐           ┌──────▼──────┐
    │ Glasses    │           │ R1 Ring     │
    │ (BLE)     │           │ (BLE,       │
    │            │           │  optional)  │
    └────────────┘           └─────────────┘
```

The glasses connect to the mobile or desktop app via Bluetooth Low Energy (BLE). The app handles all intelligence — wake word detection, LLM processing, skill execution — and pushes display content to the glasses through the adapter layer. Audio output routes through the phone's current audio device (earbuds, phone speaker, or connected speaker).

### BLE Communication

The glasses communicate using standard BLE 5.x with GATT services. No proprietary pairing gate or certificate checking exists on the Even G2 — any BLE client can connect. The Even Realities companion app is **not required** for day-to-day use. The community has fully documented the BLE protocol including the authentication handshake, CRC-16/CCITT framing, dual-channel design (content and rendering characteristics), and protobuf payloads.

**The official Even Realities companion app is kept installed solely for firmware updates and initial device setup (waking the glasses from shipping mode).** Day-to-day, the assistant's mobile or desktop app owns the BLE connection exclusively.

The glasses only hold one BLE connection at a time. Whichever app currently holds the connection has exclusive control.

**BLE libraries:**

| Platform | Library |
|---|---|
| Mobile (MAUI) | Plugin.BLE |
| Windows desktop | Windows.Devices.Bluetooth |
| Mac desktop | CoreBluetooth (via .NET binding) |
| Linux desktop | BlueZ (via .NET binding) |

### Display Adapter Layer

The adapter layer abstracts glasses communication behind a common interface (`IGlassesAdapter`). Each adapter handles the specifics of a particular device's BLE protocol — service discovery, authentication, content formatting, and display commands.

The first and only launch adapter targets the Even Realities G2. Additional adapters (e.g. Vuzix Z100) are added on request when users raise the need.

The adapter exposes capabilities so the system can query what a particular device supports:

- Display resolution and format
- Microphone availability
- Touch input availability
- Head gesture support
- Companion ring support
- Speaker availability (none on Even G2)

Features that depend on unsupported capabilities degrade gracefully — if a future adapter targets glasses with built-in speakers, audio responses can route through them. If speakers are absent (Even G2), audio routes through the phone/earbuds.

### HUD Content Tiers

The display is optimised for glanceable information. Content is organised into five tiers based on how naturally it fits the green monochrome, 640x350 HUD.

#### Tier 1 — Always Present (when HUD is active)

- Time and date
- Compact status symbols: glasses battery, Bluetooth connection, music playing indicator

Status symbols are rendered as small icons at the edge of the display, taking minimal vertical space. They are shown constantly whenever the HUD is active, not just on the dashboard.

#### Tier 2 — Push Notifications (appear briefly, dismiss on gesture or timeout)

- Message previews (sender + first line)
- Incoming call display
- Timer and reminder alerts
- Calendar event reminders (configurable lead time)
- Weather alerts
- System alerts ("server offline", "glasses battery low")
- Skill notifications following the existing urgency system (Section 12)

#### Tier 3 — Active Use (temporary overlay, replaces dashboard while in use)

- Navigation turn-by-turn arrows, street names, and distance
- AI assistant short text responses (1-3 lines)
- Real-time translation overlay
- Music — now playing info on track change (3-5 seconds, then dismisses back to dashboard). A small music note status icon persists in the tier 1 symbols while music is playing. Tap or voice command to show now playing again on demand.

#### Tier 4 — Hybrid Delivery (text on HUD + audio through earbuds/phone)

- Longer AI responses — summary on HUD, full response spoken
- Email previews — sender and subject on HUD, body read aloud if requested
- Message previews — preview on HUD, full message spoken

The assistant selects the delivery method based on response length and context. Short confirmations and alerts are text-only on the HUD. Longer responses use the hybrid approach.

#### Tier 5 — Not Displayed on Glasses (other clients only)

- Full email bodies
- Complex formatted content
- Anything requiring images or colour
- Long-form text (web search results, document reading)

### Navigation

The glasses receive turn-by-turn navigation data from the Google Maps Navigation SDK running in headless mode on the phone (see Section 15 for the full navigation architecture). The Even Realities proprietary mapping is not used.

On mobile, the Navigation SDK provides a real-time data feed — manoeuvre type, road names, distance, ETA, lane guidance — all calculated on-device after the initial route fetch with no additional API calls during navigation. The glasses adapter pushes this data to the HUD: a directional arrow, street name, distance to the next turn, and remaining ETA. Multi-destination routes update the HUD as each waypoint is reached.

When offline, the assistant falls back to OSRM with downloaded map data and pushes basic turn-by-turn instructions to the glasses (no real-time re-routing or traffic).

Navigation supports walking, cycling, and driving modes. The driving safety lockout (see below) applies — if the user is detected as driving, visual navigation pauses and falls back to audio-only directions through earbuds/phone. Passengers see navigation on the HUD.

### Audio Delivery

The Even G2 has microphones but no speakers. Audio output routes through the phone's current audio device:

- **Earbuds paired to phone** — TTS and audio responses play through earbuds (recommended for full experience)
- **Phone speaker** — TTS plays from the phone if no earbuds are connected
- **Desktop speakers** — if glasses are paired to the desktop

The assistant uses a hybrid delivery model. Short responses (confirmations, alerts, single-line answers) are displayed as text on the HUD with no audio. Longer responses (AI answers, message readouts, email bodies) are summarised on the HUD and spoken in full through the audio output.

If a future glasses adapter targets a device with built-in speakers, the adapter's capability declaration allows audio to route through the glasses directly.

### Input Methods

#### Voice (primary)

Voice is the primary input method. The glasses' microphones stream audio over BLE to the connected device (phone or desktop), where openWakeWord processes it for wake word detection. Once the wake word is recognised, the audio stream is captured for the voice command and routed through the assistant's normal STT pipeline (Whisper).

Wake word detection runs via the glasses mic — the phone may not be nearby (in a bag, across the room). The glasses are the microphone source; the connected device is the processing engine.

Active, passive, and silent listening modes work the same as on the desktop (Section 9):

| Mode | Behaviour |
|---|---|
| Passive | Glasses mic listens for wake word only. No transcription, no storage. |
| Active | Glasses mic listens and the assistant responds to everything. |
| Silent | Microphones disabled. No audio processing. Activated via touch gesture or voice command. |

#### Touch Strips

The Even G2 has capacitive touch strips on both temples supporting single tap, double tap, long press, and knock gestures. Each gesture is mapped to a configurable action:

| Gesture | Default Action |
|---|---|
| Single tap | Dismiss notification / select |
| Double tap | Go back / cancel |
| Long press | Activate assistant (alternative to wake word) |
| Knock | Cycle through active content |

Gesture-to-action mapping is configurable in the settings panel.

#### Head Gestures

The G2's IMU supports head gesture recognition:

| Gesture | Default Action |
|---|---|
| Head up | Wake HUD / show dashboard |
| Nod | Confirm / yes / dismiss |
| Shake | Decline / no / cancel |

Nod and shake enable hands-free confirmation when voice is not appropriate (meetings, quiet environments). Sensitivity is configurable in settings to reduce false positives from natural head movement.

#### R1 Companion Ring (optional)

The Even R1 ring connects via BLE and acts as a controller:

| Gesture | Action |
|---|---|
| Single tap | Confirm / select |
| Double tap | Go back / dismiss |
| Swipe (hold + slide) | Scroll through content |

The ring is fully supported out of the box but nothing requires it — every ring action has a voice, touch, or head gesture equivalent.

Ring battery level and charging state are displayed alongside the glasses in the settings panel and covered by the same proactive low-battery alerts.

### Display Behaviour

**Default: wake to show.** The HUD is dark when not in use. It activates on user interaction (head-up gesture, tap, voice command, wake word, or ring input) and returns to dark after a configurable timeout (default 10 seconds) once the interaction completes.

**Always-on option:** A toggle in settings. When enabled, the tier 1 dashboard (time, date, status symbols) is shown constantly while the glasses are worn. Notifications and active content overlay on top.

**Tied to assistant active state:** Active features (navigation, translation, conversation) keep the HUD alive for their duration. When the feature ends (conversation finishes, navigation arrives, translation stops), the HUD remains for the configured timeout period and then goes dark.

**Wear detection:** The HUD does not activate automatically when the glasses are put on. Wear detection's role is notification routing (are the glasses being worn?) and power saving (don't process BLE display commands if the glasses are on a desk), not triggering the display.

### Authentication and Guest Access

The glasses use speaker recognition to determine who is wearing them. Three states govern access:

| State | Trigger | What's Available |
|---|---|---|
| **Locked** | Glasses put on, no voice recognised yet | HUD dark until user engages. On engagement: nothing (guest toggle off) or guest HUD (guest toggle on). |
| **Unlocked** | Primary user voiceprint confirmed | Full experience — all tiers, notifications, personal data. |
| **Guest** | Unrecognised voice, guest toggle on | Time, date, weather, general Q&A only. No personal data, no notifications, no messages, no calendar. |

**Unlock flow:**

1. Glasses are put on — wear detection fires, HUD stays dark
2. User engages (gesture, tap, voice, ring)
3. If no voice yet — guest HUD only (if guest toggle is on) or nothing (if guest toggle is off)
4. User speaks — speaker recognition runs against known voiceprints
5. Primary user confirmed — glasses unlock to full access
6. Glasses remain unlocked until wear detection drops (glasses removed)
7. Glasses put back on — cycle resets to locked

Guest access is controlled by a toggle in settings (default off). When off, unrecognised wearers get no HUD content at all. When on, they get the limited guest experience — same concept as the "unknown" tier in the identity hierarchy (Section 10).

### Driving Safety

The glasses include a visual lockout when the wearer is detected as driving. Audio responses continue through earbuds or the phone — only the HUD display is affected.

**Detection:** The phone's GPS detects sustained speed above a configurable threshold (default 15 mph / 24 km/h) for 10+ seconds. The HUD suspends, showing a brief "Driving detected — HUD paused" message before going dark.

**Passenger override:** The user can say "I'm a passenger" or tap a toggle in the app to re-enable the HUD. The override persists until speed drops to zero for 2+ minutes (journey ended), then resets automatically.

**Audio continues:** Voice interaction works normally during the lockout. The assistant can still hear commands via the glasses mic and respond via earbuds/phone speaker. Only the visual display is suspended.

**Configurable:** The speed threshold and detection duration are adjustable in settings. The lockout can be disabled entirely (at the user's own risk).

**Liability:** The documentation includes a safety disclaimer that the driving lockout is a safety feature, not a guarantee, and the user is responsible for safe driving. This mirrors Even Realities' own policy of not supporting driving navigation for safety reasons.

### Notification Delivery

When the glasses are worn (wear detection active) and unlocked (primary user confirmed), they become the **primary notification channel**, replacing the phone screen or desktop UI notification for present states.

| Urgency | HUD Behaviour | Audio Behaviour |
|---|---|---|
| Low | HUD notification (text, auto-dismiss after configured duration) | Silent |
| Medium | HUD notification (text, persists until dismissed) | Optional chime through earbuds/phone |
| High | HUD notification + wakes display if dark | Spoken through earbuds/phone |
| Critical | HUD notification + wakes display + persists | Spoken immediately through earbuds/phone |

**When glasses are not worn** (wear detection inactive, glasses in case, BLE disconnected, or locked/guest state), the existing notification escalation paths from Section 12 apply as if the glasses do not exist. Notifications are not pushed to glasses sitting on a desk.

**Focus mode:** Low urgency is queued. Medium shows on HUD only (no audio). High wakes display with audio. Critical overrides everything.

### Companion Ring — Health Data

The R1 ring tracks heart rate, HRV, SpO2, body temperature, steps, calories, and sleep. At launch, the ring is supported as an **input device and for battery/connection monitoring only**. Health data reading and integration is a possible future enhancement, not a launch commitment.

**Launch scope:**

- Ring battery level, charging state, and connection status in the settings panel
- Ring input gestures (tap, double tap, swipe)
- Proactive low-battery alerts (same as glasses)

**Possible future enhancement (if added):**

- Read and display raw health metrics on request ("What's my heart rate?")
- Store health data locally for the user's reference
- User-configured threshold alerts ("Tell me if my heart rate goes above 120")
- Sleep data as context for assistant tone adjustment (shorter briefings after poor sleep)
- Step count and activity available as skill data

If health features are added, the documentation will include a disclaimer that the assistant is not a medical device and health data is for informational purposes only.

### Offline Behaviour

The glasses inherit the offline behaviour of whichever device they are connected to. They add no new offline complexity — they are a display layer on top of the connected device's capabilities.

| Scenario | Experience |
|---|---|
| Glasses connected, phone has internet | Full experience. All tiers, all features. |
| Glasses connected, phone has no internet | Tier 1 dashboard works (time from phone clock). Calendar events already synced are available. Navigation works with downloaded OSRM map data. Reminders and timers work. AI limited to on-device model. Translation limited to local LibreTranslate if running. Online skills offer to queue (existing behaviour from Section 8). |
| Glasses lose BLE connection | Glasses go dark. Assistant continues on phone/desktop as normal. On BLE reconnect, glasses pick up current state — no replay of missed notifications. |
| Desktop edition, no internet | Same as phone scenario but with the desktop's larger offline model and locally running OSRM. |

### Privacy

**No camera** removes the primary privacy concern associated with smart glasses.

**Microphone:** The glasses have four microphones that stream audio to the connected device for wake word detection and voice commands. In passive mode, audio is processed locally for wake word detection only — no transcription, no storage, nothing sent to the server. The mic can be fully disabled via silent mode (touch gesture or voice command).

**Display visibility:** Waveguide displays are only visible to the wearer. Someone standing next to the user cannot see what is on the HUD.

**No ambient audio storage:** The assistant never stores ambient audio. Only wake word detection audio and active command audio are processed. If translation or transcription features are active, they have clear on/off states and do not persist raw audio after the session ends.

**Social acceptability:** No camera means the glasses look like normal glasses — no social stigma, no visible recording light needed. The only external tell is a faint green glow on the lens when the HUD is active, which is barely visible to others.

### Relationship to Existing Clients

The glasses are an **addon module** — the same pattern as the hologram. They are not a standalone client or a separate edition. The glasses adapter is built into both the mobile app and the desktop app as an optional module.

The glasses function as a **roaming display layer** that follows the user across contexts:

| Context | Glasses Paired To | Experience |
|---|---|---|
| At home, at desk | Desktop (BLE) | Full experience alongside the hologram (if present) |
| Out and about | Phone (BLE) | Full experience alongside Telegram |
| In the car (passenger) | Phone (BLE) | Full experience, visual navigation on HUD |
| In the car (driver) | Phone (BLE) | Audio only, HUD locked out |

**Handover:** The glasses connect to whichever paired device is in BLE range. If both the phone and desktop are in range, the user configures a priority device in settings (default: phone). When the priority device is out of range, the glasses connect to the other. When the priority device returns to range, the glasses switch back. Content continuity is seamless — the glasses pick up the current state of whichever assistant they connect to.

**Relationship to hologram:** When the glasses are connected to the same assistant as the hologram (both on the desktop), the hologram takes priority for visual output. Notifications may still appear on the glasses HUD, but the hologram is the richer visual experience at the desk. When the glasses are connected to a different assistant (e.g. glasses on phone connected to a different assistant than the desktop), neither takes priority — they are independent.

### Assistant Routing (Multi-Device Only)

By default, the glasses talk to whichever assistant the mobile or desktop app is connected to. However, the mobile app supports **routing glasses traffic to a different assistant** on the home network. This allows a user to interact with one assistant through the phone and a completely different assistant through the glasses — simultaneously.

**Why this matters:** A user might want a dedicated glasses-only assistant with a different personality, or might want their glasses to connect to a sibling assistant running on a different machine while their phone stays connected to their primary assistant.

#### How It Works

The mobile app maintains two independent connections when assistant routing is configured:

1. **Phone connection** — the app's own connection to its primary assistant (used for phone screen interaction, Telegram, voice through phone mic/speaker)
2. **Glasses connection** — a separate network connection to a different assistant's server, carrying all glasses BLE traffic (mic audio, HUD content, notifications, input gestures)

Both connections are fully independent. The phone assistant and the glasses assistant are unaware of each other (unless they are siblings on the family server, in which case they have the normal sibling relationship).

#### Assistant Discovery

The mobile app discovers available assistants through two sources:

| Source | What It Provides |
|---|---|
| Family server (if configured) | Sibling roster — all assistants created by the same creator, with names, personalities, and server addresses |
| Hunter Industries API (if registered) | Assistant registry — all registered assistants, with names and connection details |

The glasses routing setting shows a list of available assistants from these sources. The user selects which one the glasses should route to.

#### Configuration

The glasses routing setting is in the glasses section of the mobile app settings:

| Setting | Options |
|---|---|
| Glasses assistant | Same as phone (default) / [list of available assistants on the network] |

When set to "Same as phone," the glasses work exactly as described in the rest of this section — they are a display layer for the phone's assistant. When set to a specific assistant, the mobile app opens a separate network connection to that assistant's server and routes all glasses BLE traffic through it.

#### Desktop Limitation

The desktop app does **not** support assistant routing. A desktop install is tied to a specific assistant — the personality files, model, configuration, and data are all local to that install. To connect a different assistant to the desktop, a separate install is required. Glasses connected to the desktop always talk to that desktop's assistant.

#### Requirements

- Multi-Device edition only (requires network access to other assistants' servers)
- The target assistant must be running and reachable on the network
- The target assistant's server must be configured to accept connections from the mobile app (API key or mutual TLS, as defined in Section 26)
- Family server or Hunter Industries API must be configured for assistant discovery

#### Offline Behaviour

If the target assistant's server is unreachable (offline, network issue), the glasses fall back to the phone's own assistant until the target server comes back. A brief HUD notification informs the user: "[Assistant name] is unreachable — using [phone assistant name]." When the target server comes back online, the glasses switch back automatically.

### Edition Compatibility

| Edition | Glasses Support | Notes |
|---|---|---|
| Desktop Only (Edition A) | Yes | Glasses pair to desktop Bluetooth. Works within BLE range (~10m). |
| Multi-Device (Edition B) | Yes | Glasses pair to phone (primary) and/or desktop. Full experience at home and on the go. Handover between devices. |
| Car edition | No direct support | The car edition has no glasses adapter. A passenger's glasses are handled by their phone. |
| Droid edition | No interaction | The droid is its own thing. |

### Relationship to USB Hardware

The Halo AI crystal's LED state system (Section 17) uses the shared "companion device active" indicator defined in Section 34. When the glasses are connected to the same assistant stored on the USB, the crystal shows the companion device active LED state — the same indicator used for the hologram or any future addon that takes ownership of visual output or notification delivery.

### Device Management

The Even Realities companion app is kept installed but is only used for:

- **Initial setup:** Waking the glasses from shipping mode (one-time)
- **Firmware updates:** OTA updates to the glasses and R1 ring firmware (a few times per year)

For firmware updates, the user disconnects the assistant's app, opens the Even Realities app, performs the update, then reconnects. This is a minor inconvenience that occurs infrequently.

**All other device management is handled by the assistant's mobile or desktop app:**

- Battery levels (glasses left arm, right arm, case, R1 ring)
- Charging state
- Connection status and signal strength
- Firmware version (display only)
- Display settings (brightness, distance, height, auto-brightness)
- Gesture configuration
- Wear detection toggle
- Silent mode
- All settings listed in the settings table below

### Legal Considerations

- Same overarching licence as the main project (proprietary source-available, Section 13)
- No trademarked names in the product — the addon is not branded with any manufacturer's name
- The Even Realities name and G2 branding are referenced in documentation as supported hardware only, not in the product name
- BLE protocol usage is clean — Even Realities ships an open-source BSD-2-Clause demo app and the community has documented the protocol under open-source licences (Apache 2.0 for MentraOS). No reverse engineering of proprietary software.
- **Driving safety disclaimer:** The documentation includes a liability notice that the driving lockout is a safety feature, not a guarantee, and the user is responsible for safe driving
- **No camera means no image capture privacy concerns** — the primary legal concern associated with smart glasses (recording others without consent) does not apply

### Settings

The glasses addon adds the following to the settings UI in both the mobile and desktop apps:

| Section | Setting | Description |
|---|---|---|
| Glasses | Device info | Glasses serial number, firmware version, R1 ring info if paired |
| Glasses | Battery detail | Left arm, right arm, case, ring — percentage, charging state |
| Glasses | Connection | Status, signal strength, connected device (phone/desktop), priority toggle |
| Glasses | Display brightness | Manual level + auto-brightness toggle |
| Glasses | Display distance | Near / Mid / Far |
| Glasses | Display height | Vertical position slider |
| Glasses | Head-up angle | 0-60° slider + calibrate button |
| Glasses | Display timeout | 3s / 5s / 10s / 15s |
| Glasses | Display mode | Wake to show (default) / Always on |
| Glasses | Wear detection | On / Off |
| Glasses | Silent mode | On / Off |
| Glasses | Guest usage | Allow guests to use glasses (default off) |
| Glasses | Driving safety | Speed threshold, auto-lockout on/off |
| Glasses | Glasses assistant | Same as phone (default) / list of available assistants on the network (Multi-Device mobile app only) |
| Glasses | Gestures — touch | Single tap, double tap, long press, knock — action assignment |
| Glasses | Gestures — head | Nod, shake — action assignment, sensitivity slider |
| Glasses | Ring input | Tap, double tap, swipe — action assignment (visible only when ring paired) |
| Voice | Glasses mic | Wake word via glasses mic on/off, mic source priority |
| Notifications | Glasses as primary | Use glasses for notifications when worn (default on) |

### Addon Roadmap

Separate from the core roadmap, edition roadmaps, and hologram addon roadmap. Each phase is self-contained and shippable.

| Phase | Content |
|---|---|
| G1 — BLE Foundation | IGlassesAdapter abstraction, Even G2 BLE adapter implementation (Plugin.BLE for mobile, Windows.Devices.Bluetooth for desktop), device discovery, pairing, connection management, wear detection, battery/charging/connection status reading, R1 ring connection and battery status |
| G2 — Settings Panel | Glasses section in mobile and desktop settings UI — device info, battery detail (glasses left/right/case, ring), display brightness, display distance/height, head-up angle, auto display timeout, wear detection toggle, gesture configuration, silent mode, connection priority (phone vs desktop) |
| G3 — HUD Display | Tier 1 persistent status symbols (time, date, battery, Bluetooth, music indicator), wake-to-show default with always-on option, display timeout logic tied to assistant active state, compact status symbol rendering |
| G4 — Notifications | Tier 2 push notifications over BLE, urgency-based display behaviour (auto-dismiss/persist/wake display), wear detection gating (only push when worn), integration with existing notification escalation system |
| G5 — Voice Input | Glasses mic audio streaming over BLE, wake word detection on connected device (openWakeWord), voice command capture (Whisper STT), active/passive/silent listening modes |
| G6 — Navigation | Google Maps Navigation SDK integration (headless mode, native binding libraries for MAUI), real-time turn-by-turn data feed push to HUD (directional arrows, street names, distance, ETA), multi-destination waypoint support, Google Maps app launch with route on phone, Android Auto/CarPlay detection and car display prompt, OSRM offline fallback, walking/cycling/driving modes |
| G7 — Input Methods | Touch strip gesture handling (single tap, double tap, long press, knock), head gesture recognition (nod, shake, head-up) with configurable sensitivity, R1 ring input (tap, double tap, swipe scroll), configurable gesture-to-action mapping |
| G8 — Active Features | Tier 3 content — AI assistant short text responses on HUD, real-time translation overlay (LibreTranslate/DeepL), music now playing on track change with persistent status icon |
| G9 — Hybrid Delivery | Tier 4 content — audio routing for longer responses through earbuds/phone, summary on HUD with full response spoken, email/message preview + spoken readout |
| G10 — Driving Safety | Speed-based GPS auto-lockout (visual only — audio continues), manual "I'm a passenger" override (voice + app toggle), auto-reset on journey end (speed zero for 2+ minutes), configurable speed threshold and detection duration |
| G11 — Handover | BLE connection handover between mobile and desktop apps, priority device configuration, seamless content continuity on device switch, automatic fallback when priority device is out of range |
| G12 — Assistant Routing | Assistant discovery via family server/Hunter Industries API, glasses routing setting in mobile app, dual connection management (phone assistant + glasses assistant simultaneously), fallback to phone assistant when target is unreachable, automatic reconnect |
