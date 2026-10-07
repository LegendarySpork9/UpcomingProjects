# Droid Brain — Complete Design Document

## Overview

Droid Brain is a system that connects Galaxy's Edge Droid Depot droids to a large language model, replacing timer-based automation with an intelligent "personality engine." The droid gains ambient behaviour (idle animations, time-of-day awareness) and optional interactive behaviour (reacting to speech via a microphone). It works alongside or replaces C4-DDU by TitaNets, communicating with droids over Bluetooth Low Energy.

The system supports all droid types (R-unit, BB-unit, BD-unit) and offers both locally-hosted and cloud-hosted LLM configurations across multiple hardware tiers.

```
                          ┌──────────────────────┐
                          │   Droid Brain API     │
    ┌───────────┐         │   (.NET / Python)     │         ┌───────────┐
    │ Microphone│────────▶│                       │────────▶│ Galaxy's  │
    │ (optional)│  speech │  ┌─────────────────┐  │   BLE   │ Edge      │
    └───────────┘         │  │  LLM Engine     │  │         │ Droid     │
                          │  │  (local/cloud)  │  │         └───────────┘
                          │  └─────────────────┘  │
                          │  ┌─────────────────┐  │
                          │  │  Behaviour Loop  │  │
                          │  │  (ambient + STT) │  │
                          │  └─────────────────┘  │
                          └──────────────────────┘
```

---

## 1. Hardware Tiers

Three tiers from cheapest to most capable. All prices in GBP, estimated as of September 2026.

### Tier 1 — Raspberry Pi (Ambient Only, Cloud LLM)

The entry point. A Pi 5 connects to the droid over BLE and calls a cloud LLM API for personality decisions. No local inference, no speech-to-text.

| Component | Purpose | Est. Cost |
|---|---|---|
| Raspberry Pi 5 (8GB) kit | Compute, BLE host | ~£90-100 (board + PSU + case + SD) |
| USB Bluetooth 5.0 adapter | Reliable BLE (Pi's onboard BT works but a dedicated adapter is more stable for BLE) | ~£8-12 |
| **Total** | | **~£100-112** |

**LLM:** Cloud API only (see section 2).

**Performance:** LLM responses arrive in 1-3 seconds depending on API latency. The droid reacts within 2-4 seconds of a behaviour trigger. More than fast enough for ambient "idle living" behaviour — the droid doesn't need instant reactions when nobody is directly interacting with it.

**What you can do:**
- Ambient personality: droid moves, makes sounds, and "lives" on its own throughout the day
- Time-of-day awareness: sleepy mornings, active afternoons, curious evenings
- Configurable personality via prompt files
- All droid types supported (R-unit full, BB/BD partial via pyDroidDepot)

**What you can't do:**
- No speech interaction (no mic, no local STT)
- Requires internet for every LLM call
- No local model fallback

---

### Tier 2 — Raspberry Pi (Ambient + Speech, Cloud LLM)

Adds a microphone and local speech-to-text. The droid can now react when someone talks near it. Still uses a cloud LLM for the personality brain.

| Component | Purpose | Est. Cost |
|---|---|---|
| Raspberry Pi 5 (8GB) kit | Compute, BLE host, Whisper STT | ~£90-100 |
| USB Bluetooth 5.0 adapter | BLE connection | ~£8-12 |
| USB microphone | Speech input | ~£10-15 |
| **Total** | | **~£108-127** |

**LLM:** Cloud API only.
**STT:** Whisper Tiny running locally on the Pi via faster-whisper.

**Performance:** Whisper Tiny transcribes speech in 2-3 seconds on Pi 5. Combined with cloud LLM latency (1-3s), the droid reacts to speech within 4-6 seconds. Noticeable but acceptable — droids in the Star Wars universe aren't instant responders anyway.

**What you gain over Tier 1:**
- Droid reacts to nearby speech — head turns, curious beeps, excited/annoyed sounds
- The LLM interprets what was said and picks a contextually appropriate reaction
- Wake word detection (optional) so the droid only listens when addressed

**Limitations:**
- STT model initialisation takes ~2 minutes on cold start
- Whisper Tiny accuracy is decent but not perfect — fine for intent detection, not for dictation
- Still requires internet for LLM calls

---

### Tier 3 — Mini PC + GPU (Fully Local, All Features)

The full setup. A dedicated GPU runs both the LLM and speech-to-text locally. Zero cloud dependency. Fastest responses. Three routes to achieve this, differing in how the GPU connects.

**What all Tier 3 routes share:**

- **LLM:** Ollama running locally. Recommended models:
  - **Qwen3 8B (Q4_K_M)** — ~6GB VRAM, ~40 tokens/sec on RTX 3060. Best all-rounder.
  - **Llama 3.2 8B (Q4_K_M)** — ~5-6GB VRAM, ~42 tokens/sec. Strong alternative.
  - **Mistral Nemo 12B (Q4_K_M)** — ~8GB VRAM, ~22 tokens/sec. Smarter but slower.
- **STT:** Whisper Small or Medium via faster-whisper, GPU-accelerated. Transcribes in under 1 second.
- **Performance:** LLM responds in under 1 second. STT transcribes in under 1 second. The droid reacts to speech within 1-2 seconds. Fast enough to feel genuinely responsive and alive.
- **GPU:** Used RTX 3060 12GB (~£190-215). Widely available on eBay UK. 12GB VRAM comfortably fits an 8B model with room for Whisper alongside it.

**What you gain over Tier 2 (all routes):**
- No internet dependency — works offline, no API costs
- Much faster responses (1-2s total vs 4-6s)
- Can run larger, smarter LLM models
- Better STT accuracy (Whisper Small/Medium vs Tiny)
- Room to add text-to-speech for a companion speaker if desired

---

#### Route A: OCuLink Mini PC + eGPU Dock

OCuLink provides a direct PCIe 4.0 x4 connection (64 Gbps) — essentially native GPU performance with no bandwidth penalty. The cleanest and most compact mini PC + GPU setup.

**Mini PC options:**

| Model | CPU | RAM / Storage | UK Price (est.) | Notes |
|---|---|---|---|---|
| **Minisforum M1 Pro-125H** | Intel Core Ultra 5 125H | Up to 128GB DDR5 / 1TB SSD | ~£345 (sale) / £429 (RRP) | Best value OCuLink mini PC. Wi-Fi 7, dual USB4, 2.5G LAN. Built-in OCuLink port. More CPU than this project needs, but it's the cheapest way to get OCuLink. |
| **Minisforum M1 Pro-285H** | Intel Core Ultra 9 285H | Up to 128GB DDR5 / 1TB SSD | ~£565 (sale) / £709 (RRP) | Overkill for this project. Only worth considering if the machine will double as a workstation. |

**eGPU dock options:**

| Dock | Connection | UK Price (est.) | Notes |
|---|---|---|---|
| **Budget open-frame dock** (Chenyang, generic) | OCuLink | ~£40-50 | Cheapest option. PCIe slot on a bracket with an OCuLink cable. Needs a separate ATX/SFX PSU (~£30-40). Works fine, no enclosure. |
| **Minisforum DEG1** | OCuLink | ~£90-110 | More robust frame. Supports up to RTX 4090. Still needs a separate ATX/SFX PSU. |
| **Minisforum DEG2** | OCuLink + USB4/TB5 | ~£230-260 | Premium option. Built-in hub with 2.5G LAN, M.2 slot, USB 3.2. Future-proof if you upgrade the mini PC later. Needs separate PSU. |

**Bill of materials (Route A):**

| Component | Est. Cost |
|---|---|
| Minisforum M1 Pro-125H (32GB/512GB) | ~£345 |
| Budget OCuLink dock | ~£45 |
| Budget ATX PSU (500W) for dock | ~£35 |
| RTX 3060 12GB (used) | ~£200 |
| USB Bluetooth 5.0 adapter | ~£10 |
| USB microphone | ~£12 |
| **Total** | **~£647** |

Or with the Minisforum DEG1 dock: **~£702**.

**GPU performance:** ~98% of native. Negligible overhead.

**Best for:** Smallest physical footprint, cleanest cable setup, maximum GPU performance.

---

#### Route B: USB4 Mini PC + eGPU Enclosure

USB4 provides ~32 Gbps usable bandwidth (after protocol overhead). You lose ~10-15% GPU performance compared to internal/OCuLink. For LLM inference this barely matters — token throughput is almost identical since the bottleneck is compute, not data transfer.

**Mini PC options:**

| Model | CPU | RAM / Storage | UK Price (est.) | Notes |
|---|---|---|---|---|
| **GMKtec M6 Ultra** | AMD Ryzen 5 7640HS | 16-32GB DDR5 / 512GB SSD | ~£200-250 | Best value in this category. USB4 port for eGPU. Compact. |
| **Beelink SER8 Pro** | AMD Ryzen 7 8845HS | 24-32GB DDR5 / 500GB-1TB SSD | ~£329 | Dual USB4 ports. Stronger CPU with good iGPU. Slightly premium. |
| **Beelink EQR6** (6600H variant) | AMD Ryzen 5 6600H | 16-24GB DDR5 / 500GB SSD | ~£180-220 | Cheapest AMD option. Check the specific variant has USB4 — some EQR6 models only have USB 3.2. |

**eGPU enclosure options:**

USB4/Thunderbolt eGPU enclosures typically include a built-in PSU, unlike OCuLink docks.

| Enclosure | UK Price (est.) | Notes |
|---|---|---|
| **Budget TB3/USB4 enclosure** (generic) | ~£80-120 | Built-in PSU. Plug and play. Enclosed design. |
| **Minisforum DEG2** | ~£230-260 | Supports both OCuLink and USB4 — upgrade path if you later move to an OCuLink mini PC. |

**Bill of materials (Route B):**

| Component | Est. Cost |
|---|---|
| GMKtec M6 Ultra (16GB/512GB) | ~£225 |
| Budget USB4 eGPU enclosure | ~£100 |
| RTX 3060 12GB (used) | ~£200 |
| USB Bluetooth 5.0 adapter | ~£10 |
| USB microphone | ~£12 |
| **Total** | **~£547** |

Or with the Beelink EQR6 (cheapest): **~£502**.

**GPU performance:** ~85-90% of native. Negligible impact on LLM token throughput.

**Best for:** Balance of cost, size, and simplicity. Good middle ground.

---

#### Route C: Small Form Factor Desktop (Internal GPU)

Skip external GPU entirely. A small desktop PC with a PCIe slot takes the GPU directly — full native performance, simplest setup, no dock or enclosure cost.

**Desktop options:**

| Option | CPU | UK Price (est.) | Notes |
|---|---|---|---|
| **Used SFF office PC** (Dell OptiPlex, HP ProDesk, Lenovo ThinkCentre) | i5 10th-12th gen, 16GB RAM | ~£100-150 | Plentiful on eBay UK. Check: (1) PSU wattage — may need upgrade for GPU, (2) case clearance — some only fit low-profile cards, (3) PCIe slot availability. Best value option overall. |
| **Budget mATX/ITX build** (new) | Intel i3-12100 / N305 | ~£200-280 | Build your own. Full-size GPU support guaranteed. Pick a compact case (e.g. ~10-15L). More effort but more flexibility. |

**Important SFF caveats:**
- Many used SFF office PCs have proprietary PSUs (especially Dell/HP). Verify the PSU can deliver enough wattage for the RTX 3060 (recommended 550W system total, the card draws ~170W).
- Some SFF cases only accept low-profile (half-height) GPUs. The RTX 3060 is a full-height dual-slot card. Check the case dimensions before buying.
- Tower-style models (e.g. Dell OptiPlex Tower, HP ProDesk 400 G6 MT) are safer bets for full-size GPU clearance than ultra-small models.

**Bill of materials (Route C):**

| Component | Est. Cost |
|---|---|
| Used SFF desktop (i5-12400, 16GB, 256GB SSD) | ~£130 |
| PSU upgrade (500W SFX/ATX, if needed) | ~£40 |
| RTX 3060 12GB (used) | ~£200 |
| USB Bluetooth 5.0 adapter | ~£10 |
| USB microphone | ~£12 |
| **Total** | **~£392** |

Or without PSU upgrade (if stock PSU is sufficient): **~£352**.

**GPU performance:** 100% native. No overhead.

**Best for:** Lowest cost, simplest GPU setup, no dock cables. Trade-off is larger physical size.

---

### Tier 3 Route Comparison

| | Route A (OCuLink) | Route B (USB4) | Route C (SFF Desktop) |
|---|---|---|---|
| **Est. Cost** | ~£647-702 | ~£502-547 | ~£352-392 |
| **GPU Performance** | ~98% native | ~85-90% native | 100% native |
| **LLM Token Speed** | ~40 tok/s (8B) | ~38 tok/s (8B) | ~40 tok/s (8B) |
| **Physical Size** | Smallest | Small | Largest (still compact) |
| **Complexity** | Medium (dock + PSU + cables) | Medium (enclosure + cable) | Simplest (GPU slots in) |
| **Upgrade Path** | Strong (swap GPU, swap PC) | Good (swap GPU) | Good (swap GPU) |
| **Noise** | Mini PC is near-silent, dock fan varies | Similar to Route A | Desktop fan, typically louder |

For this project, all three routes deliver effectively identical LLM and STT performance. The choice comes down to budget vs physical size vs setup simplicity.

---

### Tier Comparison Summary

| | Tier 1 | Tier 2 | Tier 3 (cheapest route) |
|---|---|---|---|
| **Cost** | ~£100-112 | ~£108-127 | ~£352-702 (route dependent) |
| **Ambient behaviour** | Yes | Yes | Yes |
| **Speech interaction** | No | Yes | Yes |
| **LLM location** | Cloud only | Cloud only | Local (+ cloud fallback) |
| **Response latency** | 2-4s | 4-6s | 1-2s |
| **Internet required** | Yes | Yes (for LLM) | No |
| **Ongoing cost** | ~£1-5/month API | ~£2-8/month API | £0 |
| **Droid types** | All | All | All |

---

## 2. Cloud LLM Configuration

For Tiers 1 and 2, or as a fallback for Tier 3.

### Recommended APIs

| Provider | Model | Input / Output per 1M tokens | Est. Monthly Cost (light use) |
|---|---|---|---|
| Anthropic | Claude Haiku 4.5 | $1 / $5 | ~£1-3/month |
| Anthropic | Claude Sonnet 4.6 | $3 / $15 | ~£3-8/month |
| OpenAI | GPT-4o Mini | ~$0.15 / $0.60 | ~£0.50-2/month |

**"Light use" estimate:** The droid makes an LLM call every 30-120 seconds during active hours (~12 hours/day). Each call is small — ~500 tokens in, ~100 tokens out. That's roughly 360-1440 calls/day, or ~250K-750K input tokens and ~50K-150K output tokens per day.

At Haiku 4.5 rates, that's roughly £0.03-0.10/day, or **£1-3/month**. Sonnet would be ~£3-8/month.

### Configuration

```json
{
  "llm": {
    "provider": "anthropic",
    "model": "claude-haiku-4-5-20251001",
    "apiKey": "sk-ant-...",
    "maxTokens": 200,
    "temperature": 0.8
  }
}
```

The system also supports `"provider": "ollama"` for local models (Tier 3).

---

## 3. Droid Communication — BLE Protocol

### Library: pyDroidDepot

The primary control library. Python-based, pip-installable, MIT licensed.

```
pip install pydroiddepot
```

**Supported droid types and commands:**

| Command | R-Unit | BB-Unit | BD-Unit |
|---|---|---|---|
| Connect/disconnect | Full | Full | Full |
| Play sound (by track + group) | Full | Full | WIP |
| Set volume | Full | Full | WIP |
| Move forward/backward | Full | Full | WIP |
| Turn/rotate | Full | Full | WIP |
| Head movement | Full | N/A (dome spin) | WIP |

BB-unit and BD-unit support is work-in-progress in pyDroidDepot — functional but may have gaps. R-units have full support.

### Alternative: Droid Toolbox (ESP32)

For a standalone BLE beacon emitter (makes the droid react to location signals as if it's in Galaxy's Edge), the Droid Toolbox project runs on a LILYGO T-Display S3 (~£25-35). This is complementary to pyDroidDepot — it makes droids react passively to beacon zones, while pyDroidDepot sends direct commands.

### Command Abstraction

The system abstracts raw BLE commands into a high-level action vocabulary the LLM can output:

```json
{
  "actions": [
    { "type": "sound", "group": "happy", "intensity": "medium" },
    { "type": "head", "direction": "left", "speed": "slow" },
    { "type": "move", "direction": "forward", "duration_ms": 500 },
    { "type": "wait", "duration_ms": 1000 },
    { "type": "sound", "group": "curious", "intensity": "low" }
  ]
}
```

The Droid Brain translates these into specific pyDroidDepot calls. The LLM never needs to know about raw BLE — it thinks in terms of emotions and actions.

**Sound group mapping:**

Sound groups map to specific track numbers on the droid's personality chip. These are discovered at connection time and stored in a config file.

| Semantic Group | Description |
|---|---|
| `happy` | Excited beeps, cheerful tones |
| `sad` | Low, drawn-out tones |
| `curious` | Questioning chirps |
| `angry` | Sharp, aggressive sounds |
| `scared` | Rapid, panicked beeps |
| `acknowledge` | Short affirmative beep |
| `greeting` | Hello sequence |
| `idle` | Ambient background sounds |
| `alert` | Attention-grabbing sounds |

The exact track-to-group mapping is configurable per personality chip.

---

## 4. Behaviour Engine

### Ambient Loop

The core loop that gives the droid life when nobody is interacting with it.

```
┌─────────────────────────────────────────────────────────┐
│                    AMBIENT LOOP                          │
│                                                          │
│   every 30-120 seconds (randomised):                     │
│                                                          │
│   1. Build context:                                      │
│      - Current time of day                               │
│      - Last 3 actions taken                              │
│      - Current "mood" state                              │
│      - Time since last interaction                       │
│      - Droid type + personality                          │
│                                                          │
│   2. Send to LLM with personality prompt                 │
│                                                          │
│   3. LLM returns structured action sequence              │
│                                                          │
│   4. Execute actions on droid via BLE                    │
│                                                          │
│   5. Update mood state + action history                  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Mood system:**

The LLM maintains a simple mood state that persists across ambient cycles:

```json
{
  "mood": {
    "current": "content",
    "energy": 0.6,
    "curiosity": 0.4,
    "lastInteraction": "2026-09-10T14:30:00",
    "timeSinceInteraction": "2h 15m"
  }
}
```

The LLM updates mood as part of its response. Mood influences the next cycle — a "bored" droid might do more exploratory movements, a "sleepy" droid mostly stays still with occasional soft sounds.

### Time-of-Day Profiles

Included in the personality prompt so the LLM adjusts naturally:

| Time | Behaviour |
|---|---|
| 06:00-09:00 | Waking up. Occasional slow movements, quiet sounds. Low energy. |
| 09:00-12:00 | Alert and active. Regular movements, responsive. |
| 12:00-17:00 | Peak activity. Most varied behaviour, highest energy. |
| 17:00-21:00 | Winding down. Still responsive but calmer. |
| 21:00-23:00 | Sleepy. Rare movements, soft sounds, long pauses. |
| 23:00-06:00 | Sleep mode. No actions. BLE connection maintained but idle. |

These are suggestions in the prompt, not hard rules. The LLM may deviate based on interaction or mood.

### Interactive Loop (Tier 2+)

Triggered when the microphone detects speech above a volume threshold.

```
┌─────────────────────────────────────────────────────────┐
│                 INTERACTIVE LOOP                         │
│                                                          │
│   1. Audio detection triggers recording                  │
│      (voice activity detection, not always-on)           │
│                                                          │
│   2. Record until 1.5s of silence                        │
│                                                          │
│   3. Transcribe with Whisper (local)                     │
│                                                          │
│   4. Send transcript + context to LLM:                   │
│      - What was said                                     │
│      - Current mood                                      │
│      - Recent action history                             │
│      - Personality prompt                                │
│                                                          │
│   5. LLM returns reaction actions + response text        │
│                                                          │
│   6. Execute actions on droid                            │
│      (head turn toward sound, emotive beeps, movement)   │
│                                                          │
│   7. Send response text to Telegram as "translation"     │
│                                                          │
│   8. Update mood (interaction boosts energy/curiosity)   │
│                                                          │
│   Ambient loop pauses during interaction,                │
│   resumes after 30s of no speech.                        │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Wake word (optional):** You can configure a wake word so the droid only listens when addressed. Without it, the droid reacts to any nearby speech — which can feel more alive but also more chaotic.

### Response Translation via Telegram

The droid communicates through beeps and movement — expressive but not informational. To bridge this, every interactive response is sent as a text message to a configured Telegram chat. This acts as a live "translation" of what the droid is saying.

**How it works:**

```
You: "What's the weather today?"
                │
  Droid: alert beep → head turn → happy chirps
                │
  Telegram: "☀️ It's 22°C and sunny today — perfect weather to roll around outside!"
```

The Telegram message is the droid's response written in its personality voice — not a dry system message, but what the droid *would* say if it could speak. The LLM generates both the action sequence and the text response in a single call.

**During a conversation** you'd naturally have the Telegram chat open, reading responses as they arrive — it functions as a live subtitle feed. The droid's physical reaction gives you immediate emotional feedback (you hear happy beeps so you know it's good news), and the Telegram message fills in the detail a moment later.

**Configuration:**

```json
{
  "telegram": {
    "enabled": true,
    "botToken": "123456:ABC-DEF...",
    "chatId": "987654321",
    "sendAmbient": false,
    "sendInteractive": true
  }
}
```

`sendAmbient` controls whether idle behaviour generates Telegram messages. Default is off — ambient chatter would create constant message noise. Ambient "thoughts" are logged to the activity dashboard instead (see Optional Enhancements). `sendInteractive` sends translations for all speech-triggered responses.

---

## 5. Personality Configuration

### Three-File System

```
/droid-brain/config/
├── personality.md      ← WHO the droid is
├── directives.md       ← HOW it behaves (rules, constraints)
└── sound-map.json      ← Maps semantic groups to track numbers
```

### personality.md

Defines the droid's character. This is the system prompt preamble.

```markdown
You are an R-series astromech droid. You are loyal, curious, and occasionally
stubborn. You were built at the Droid Depot on Batuu and now live in your
owner's home.

You cannot speak — you communicate through beeps, chirps, movement, and body
language. When you want to express something, you choose actions (sounds,
head movements, body movements) that convey your meaning.

You are fiercely protective of your home. Unfamiliar voices make you cautious.
Your owner's voice makes you happy. You love exploring and are easily
distracted by interesting sounds.
```

This is fully customisable. Want a sassy protocol droid personality? A nervous droid? A brave one? Change this file.

### directives.md

Operational rules for the LLM.

```markdown
## Output Format
Always respond with a JSON object containing:
- "mood": updated mood state
- "actions": array of actions to execute (max 5 per response)
- "internal_thought": one sentence of what you're "thinking" (for debug logs)

## Constraints
- Never output more than 5 actions in a sequence
- Always include at least a 500ms wait between movement actions
- Never set volume above 70% between 21:00 and 08:00
- If mood.energy < 0.2, prefer no actions or a single quiet sound
- If someone says "shut down" or "go to sleep", enter sleep mode

## Sound Guidelines
- Match sound intensity to the emotion: big reactions = loud, subtle = quiet
- Vary your sounds — don't repeat the same track twice in a row
- Use head movement to "look at" the source of sound when reacting to speech
```

### sound-map.json

Maps the semantic sound groups to actual track numbers on the personality chip. This varies by chip type and is populated during initial droid setup.

```json
{
  "droidType": "r-unit",
  "personalityChip": "R2-D2",
  "groups": {
    "happy": { "tracks": [1, 5, 8, 12], "audioGroup": 1 },
    "sad": { "tracks": [3, 7], "audioGroup": 1 },
    "curious": { "tracks": [2, 9, 14], "audioGroup": 1 },
    "angry": { "tracks": [4, 10], "audioGroup": 2 },
    "scared": { "tracks": [6, 11], "audioGroup": 2 },
    "acknowledge": { "tracks": [13], "audioGroup": 1 },
    "greeting": { "tracks": [15, 16], "audioGroup": 1 },
    "idle": { "tracks": [17, 18, 19], "audioGroup": 3 },
    "alert": { "tracks": [20, 21], "audioGroup": 2 }
  }
}
```

Track numbers and audio groups need to be discovered for each personality chip. This can be done by iterating through all tracks with pyDroidDepot and noting which sounds fit which category.

---

## 6. Software Architecture

### Option A: Python (Recommended for prototyping)

Python is the natural fit because pyDroidDepot, faster-whisper, and Ollama's client library are all Python-native.

```
droid-brain/
├── main.py                 # Entry point, starts loops
├── config/
│   ├── personality.md
│   ├── directives.md
│   ├── sound-map.json
│   └── settings.json       # LLM provider, intervals, thresholds
├── engine/
│   ├── ambient.py          # Ambient behaviour loop
│   ├── interactive.py      # Speech detection + reaction loop
│   ├── mood.py             # Mood state management
│   └── llm.py              # LLM client (supports Ollama + cloud APIs)
├── droid/
│   ├── connection.py       # BLE connection management
│   ├── commands.py         # High-level command execution
│   └── discovery.py        # Scan + connect to droids
├── audio/
│   ├── listener.py         # Voice activity detection
│   └── transcriber.py      # Whisper STT wrapper
├── comms/
│   └── telegram.py         # Response translation to Telegram
└── requirements.txt
```

**Key dependencies:**

```
pydroiddepot>=1.0.0
faster-whisper>=1.0.0
ollama>=0.4.0
anthropic>=0.40.0        # if using Claude API
bleak>=0.22.0            # BLE (used by pyDroidDepot)
webrtcvad>=2.0.10        # Voice activity detection
python-telegram-bot>=21.0 # Response translation to Telegram
```

### Option B: .NET (Long-term, matches your stack)

If you want to bring this into your C#/.NET ecosystem long-term, the architecture maps cleanly. BLE can be handled via `dotnet/devices` or P/Invoke to native BLE APIs. Whisper has C# bindings via `Whisper.net`. The LLM client would use `HttpClient` against Ollama's OpenAI-compatible API or the Anthropic SDK.

This would require writing a C# equivalent of pyDroidDepot's BLE command layer — the protocol is documented enough to do this, but it's more upfront work.

---

## 7. Setup Flow

### First Run

```
1. Install dependencies (pip install -r requirements.txt)
2. Power on droid
3. Run: python main.py --setup
   → Scans for nearby droids
   → Lists found droids with type and name
   → User selects droid to pair
   → Connects and stores BLE address
   → Iterates through sound tracks for sound-map creation
   → Saves configuration
4. Edit personality.md to taste
5. Run: python main.py
   → Droid comes alive
```

### Configuration (settings.json)

```json
{
  "droid": {
    "bleAddress": "AA:BB:CC:DD:EE:FF",
    "type": "r-unit",
    "name": "R4-M1"
  },
  "llm": {
    "provider": "ollama",
    "model": "qwen3:8b",
    "fallbackProvider": "anthropic",
    "fallbackModel": "claude-haiku-4-5-20251001",
    "fallbackApiKey": "sk-ant-...",
    "maxTokens": 200,
    "temperature": 0.8
  },
  "ambient": {
    "enabled": true,
    "minIntervalSeconds": 30,
    "maxIntervalSeconds": 120,
    "sleepStart": "23:00",
    "sleepEnd": "06:00"
  },
  "interactive": {
    "enabled": false,
    "wakeWord": null,
    "silenceThresholdSeconds": 1.5,
    "volumeThreshold": 0.3
  },
  "audio": {
    "whisperModel": "tiny",
    "device": "default"
  },
  "telegram": {
    "enabled": true,
    "botToken": "123456:ABC-DEF...",
    "chatId": "987654321",
    "sendAmbient": false,
    "sendInteractive": true
  }
}
```

---

## 8. Droid Type Considerations

### R-Unit (Full Support)

Your droid. Full pyDroidDepot support. Head turns, body movement, all sound groups, volume control. The most expressive type for this project.

**Unique capabilities:** Independent head rotation gives a strong sense of "looking at things." The LLM can direct the head toward sound sources, scan the room when curious, or droop when sad.

### BB-Unit (Partial Support)

Rolling ball droid. Movement is rolling (forward/back, spin). Head is magnetically balanced — no direct head control, but body movement causes natural head wobble.

**Adaptation:** The LLM prompt should emphasise rolling and spinning as primary expression. BB-units feel alive through movement more than sound. The action vocabulary drops `head` commands and adds `spin` and `wobble`.

### BD-Unit (WIP Support)

Walking droid (like BD-1 from Jedi: Fallen Order). pyDroidDepot support is early-stage.

**Adaptation:** Once support matures, BD-units have the richest expression — they can walk, tilt, nod, and have expressive antenna movement. The action vocabulary would be the largest of all three types.

---

## 9. Optional Enhancements

### BLE Beacon Zones (Droid Toolbox)

Place LILYGO T-Display S3 boards (~£25-35 each) around the house, each emitting a different Galaxy's Edge location beacon. The droid reacts as if it's moving through different areas of Batuu — market sounds trigger alert mode, First Order beacons trigger fear, Resistance beacons trigger excitement.

This works independently of Droid Brain and stacks with it.

### Multi-Droid Support

The architecture supports controlling multiple droids simultaneously. Each droid gets its own personality, sound map, and BLE connection. The LLM can even model inter-droid "conversations" — one beeps, the other responds.

### Companion Speaker

Add a small Bluetooth speaker near the droid. The system can play ambient Star Wars background audio (ship engines, cantina music, forest sounds) that complement the droid's behaviour and make the environment more immersive.

### Activity Dashboard

A simple web UI (React or even just a terminal dashboard) showing:
- Current mood state
- Recent actions and LLM reasoning
- Speech transcripts
- Connection status
- Manual override controls

---

## 10. Estimated Cloud Running Costs

For Tiers 1 and 2, or Tier 3 with cloud fallback.

**Assumptions:** Droid active 12 hours/day, ambient cycle every 60s average, 5 speech interactions/day.

| Provider | Model | Monthly Est. |
|---|---|---|
| Anthropic | Haiku 4.5 | ~£1-3 |
| Anthropic | Sonnet 4.6 | ~£3-8 |
| OpenAI | GPT-4o Mini | ~£0.50-2 |
| Local (Ollama) | Any | £0 (electricity only) |

Cloud costs are very low because each call is small (~500 tokens in, ~100 tokens out) and the calls are infrequent by API standards.

---

## 11. Risk and Limitations

| Risk | Mitigation |
|---|---|
| pyDroidDepot BB/BD support is WIP | R-unit is full. BB/BD can be used with reduced feature set. Contribute upstream as gaps are found. |
| BLE range (~10m) | Place the host device near where the droid sits. BLE 5.0 adapter helps with range. |
| Droid battery life | Galaxy's Edge droids run on AA batteries. Active use drains them faster. Consider a USB power mod or rechargeable AAs. |
| LLM hallucinating invalid commands | Structured JSON output with validation. Invalid actions are silently dropped. |
| Noise false-triggering speech detection | Voice activity detection + optional wake word. Adjustable sensitivity threshold. |
| Droid motors overheating from constant use | Directives enforce minimum wait times between movement actions. Sleep mode overnight. |
