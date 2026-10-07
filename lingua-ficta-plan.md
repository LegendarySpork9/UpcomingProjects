# Lingua Ficta — Project Plan

*Latin: "fictional language"*
A free, self-hosted web app that translates English to and from constructed fictional languages.

---

## 1. Project Overview

### What
A web-based translator for fictional/constructed languages, starting with Doyle Kryptonese (Arrowverse Kryptonian). Designed to be expandable to additional fictional languages and accented speech patterns (e.g., Anna Marie/Rogue's Southern accent, Kurt Wagner/Nightcrawler's German-accented English, Star Wars languages).

### Who
Personal fan project by Toby Hunter. Free to use, open source.

### Where
Self-hosted on a Raspberry Pi 5 (8GB) at home.

### How
TypeScript/React frontend + Python/FastAPI backend. Rule-based translation engine with a curated idiom override table. No LLM dependency — pure deterministic rules engine.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────┐
│                    BROWSER                           │
│                                                     │
│  React/Next.js Frontend                             │
│  ├── Home Page (about, credits, links)              │
│  ├── Translator Page                                │
│  │   ├── Input Panel (text + context controls)      │
│  │   ├── Context Confirmation Panel                 │
│  │   ├── Output Panel (romanized + glyphs)          │
│  │   ├── Breakdown Toggle (interlinear gloss)       │
│  │   ├── Confidence Indicator                       │
│  │   └── Feedback / Idiom Submit buttons            │
│  └── Kryptonian Font (@font-face)                   │
│                                                     │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP/JSON API
┌──────────────────────▼──────────────────────────────┐
│              RASPBERRY PI 5 (8GB)                    │
│                                                     │
│  Python / FastAPI Backend                            │
│  ├── /api/translate          (POST)                 │
│  │   ├── Idiom Table Lookup                         │
│  │   ├── English Parser (sentence splitting,        │
│  │   │   POS tagging, dependency parsing)           │
│  │   ├── Rules Engine                               │
│  │   │   ├── Word Order (SVO → VSO)                 │
│  │   │   ├── Verb Conjugation (TAM suffixes)        │
│  │   │   ├── Mood Prefix Application                │
│  │   │   ├── Particle Insertion (w, ki, ni)         │
│  │   │   ├── Possession Logic                       │
│  │   │   ├── Gender/Register Vowel Shifting         │
│  │   │   ├── Quantifier/Article Harmony             │
│  │   │   ├── Proper Noun Transliteration            │
│  │   │   └── Phonetic Fallback (unknown words)      │
│  │   ├── Confidence Scorer                          │
│  │   └── Gloss Generator                            │
│  ├── /api/reverse-translate  (POST)                 │
│  │   └── Kryptonese → English pipeline              │
│  ├── /api/dictionary         (GET)                  │
│  │   └── Dictionary JSON (bundled)                  │
│  ├── /api/font-mapping       (GET)                  │
│  │   └── Romanization → Kryptonian font chars       │
│  └── /api/submit-issue       (POST)                 │
│      └── GitHub API → create issue                  │
│                                                     │
│  Data Files (bundled JSON)                           │
│  ├── dictionary.json                                │
│  ├── grammar-rules.json                             │
│  ├── idiom-table.json                               │
│  ├── font-mapping.json                              │
│  └── transliteration-rules.json                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | React + TypeScript | Modern, component-based, strong typing |
| Styling | Tailwind CSS | Rapid UI development, responsive |
| Backend | Python 3.11+ / FastAPI | Clean API framework, lightweight, future LLM-ready |
| NLP | spaCy (en_core_web_sm) | POS tagging, dependency parsing for English input |
| Font | Kryptonian8.2-Regular.otf | Bundled, loaded via @font-face |
| Hosting | Raspberry Pi 5 (8GB) | Self-hosted, nginx reverse proxy |
| VCS | Git + GitHub | Source control + issue tracking for submissions |
| CI/CD | GitHub Actions (optional) | Auto-deploy to Pi via SSH on push |

---

## 4. Features

### 4.1 Home Page

- Project name "Lingua Ficta" with Latin meaning explanation
- What the project is — a translator for constructed fictional languages
- Credit to Darren Doyle as language creator with link to kryptonian.info/doyle/
- Brief explanation of how the translator works (rules-based engine, not AI guessing)
- Language selector (initially just Kryptonese, expandable)

### 4.2 Translator Page

#### Input Panel
- Large text input area (multi-paragraph support)
- Language direction toggle: English → Kryptonese / Kryptonese → English
- Support for pasting Kryptonian font characters (mapped back to romanization)
- Advisory message if multiple speaker labels detected (e.g., "Kara:", "Alex:") suggesting separate translations for accurate gender/register

#### Context Confirmation Panel
Auto-detected with manual override for all fields:
- Speaker gender (feminine / masculine / neutral)
- Listener gender (feminine / masculine / neutral)
- Register (formal / informal-familiar)
- Relationship (family / friend / stranger)
- Tense override (past / present / future)
- Mood override (declarative / question / command / request)

Flow:
1. User enters text
2. Engine auto-detects context from English input
3. Confirmation panel appears with detected values and dropdowns
4. User confirms or adjusts
5. Translation proceeds

#### Output Panel
- Romanized Kryptonese text (copyable)
- Kryptonian glyph text rendered with @font-face (selectable, copyable)
- Confidence indicator (High / Medium / Low)
- Transliteration notices for any words not in the dictionary, explaining the phonetic transliteration treatment

#### Breakdown Toggle
- Hidden by default, "Show breakdown" toggle button
- Displays interlinear gloss table (word | gloss)
- Shows morpheme boundaries (prefix-stem-suffix)

### 4.3 Confidence Scoring

| Level | Criteria |
|-------|---------|
| High | All words found in dictionary, straightforward grammar, no idiom overrides |
| Medium | Some transliterations used, or complex sentence structures |
| Low | Idiom overrides applied, unusual sentence structure, multiple unknowns |

### 4.4 Idiom Handling

Flow for unrecognised idioms:
1. Rules engine encounters a phrase it can't map (no dictionary match, no idiom table match)
2. Warning displayed: "This idiom isn't in the database yet"
3. Override box appears — user provides a literal rephrase (e.g., "up to no good" → "scheming evil")
4. Rules engine translates the rephrased version normally
5. Separate "Submit this idiom for review" button (user-triggered, not automatic)
6. If clicked → GitHub issue created via API with:
   - Original idiom
   - User's suggested rephrase
   - Context provided
   - Tagged as "idiom-submission"

### 4.5 Feedback System

- "Report translation issue" button on output
- User-triggered GitHub issue creation
- Includes: input text, output text, context settings, user's correction/comment
- Tagged as "translation-feedback"

### 4.6 Reverse Translation (Kryptonese → English)

- Same dictionary and rules data, inverse parsing logic
- Accept romanized Kryptonese input
- Accept Kryptonian font character input (mapped back via font-mapping.json)
- Output English translation with confidence score

---

## 5. Data Structures

### 5.1 dictionary.json

```json
{
  "entries": {
    "skulev": {
      "kryptonian_script": "skulEv",
      "kryptonese": "skulev",
      "ipa": "skuˈlev",
      "definitions": [
        {
          "type": "noun",
          "english": ["enigma", "mystery", "secret"]
        }
      ],
      "variants": [],
      "related": ["/skulir/", "/skulium/"],
      "gender_variants": {},
      "notes": ""
    },
    "throniv": {
      "kryptonian_script": "Troniv",
      "kryptonese": "throniv",
      "ipa": "θɹo.niv",
      "definitions": [
        {
          "type": "v1",
          "english": ["conceal", "cover", "hide", "protect", "shelter", "shield"]
        }
      ],
      "variants": [],
      "related": [],
      "gender_variants": {},
      "notes": ""
    }
  }
}
```

### 5.2 grammar-rules.json

```json
{
  "word_order": "VSO",
  "verb_system": {
    "type1_suffixes": {
      "future": { "progressive": "i", "perfect": "ao", "simple": "u" },
      "present": { "progressive": "es", "perfect": "ehth", "simple": "odh" },
      "past": { "progressive": "as", "perfect": "uhsh", "simple": "ahzh" }
    },
    "type2_verbs": {
      "nahn": { "future": "nim", "present": "nahn", "past": "non", "english": ["be", "is", "am", "are"] },
      "sem": { "future": "shim", "present": "sem", "past": "som", "english": ["want", "desire"] },
      "tulem": { "future": "tulim", "present": "tulem", "past": "tulahm", "english": ["need"] }
    },
    "mood_prefixes": {
      "interrogative": "ta-",
      "potential": "kai-",
      "cohortative": "bah-",
      "exhortative": "le-",
      "hypothetical": "kah-",
      "conditional": "so-",
      "imperative": "kao-",
      "polite_imperative": "sokao-",
      "prohibitive": "zhao-",
      "polite_prohibitive": "sozhao-",
      "emphatic": "zhi-",
      "negation": "zha-"
    },
    "habitual_infix": "ahr",
    "subjunctive_suffix": "si"
  },
  "particles": {
    "complement": "w",
    "indirect_object": "ni",
    "instrument": "ki",
    "relative_clause_object": "ahm",
    "subject_relative": "zw",
    "object_relative": "to",
    "passive": "ton"
  },
  "pronouns": {
    "1sg": { "neutral": "khuhp", "feminine": "khap", "masculine": "khahp" },
    "1pl": { "neutral": "kryp", "feminine": "krep", "masculine": "krop" },
    "2nd": { "neutral": "rraop", "feminine": "rrip", "masculine": "rrup" },
    "3rd_personal": { "neutral": "zhehd", "feminine": "zhed", "masculine": "zhod" },
    "3rd_animate": { "neutral": "ghao", "feminine": "ghi", "masculine": "ghu" },
    "3rd_inanimate": { "neutral": "gehd" }
  },
  "possession": {
    "genitive_particle": "im",
    "inalienable_particle": "i",
    "alienable": {
      "process": [
        "Take pronoun base",
        "Drop final consonant",
        "Append definite article tiv",
        "Voice t based on dropped consonant (voiced→d, voiceless→t, vowel→d)",
        "Article vowel harmonizes with possessed noun quantifier"
      ]
    }
  },
  "quantifier_suffixes": {
    "singular": "",
    "plural": "o",
    "many": "u",
    "most": "use",
    "all": "uju",
    "few": "ah",
    "none": "ahjah"
  },
  "article_harmony": {
    "singular": "tiv",
    "plural": "tov",
    "many_most_all": "tuv",
    "few_none": "tahv"
  },
  "gender_vowel_rows": [
    { "feminine": "i", "neutral": "ao", "masculine": "u" },
    { "feminine": "e", "neutral": "eh", "masculine": "o" },
    { "feminine": "a", "neutral": "ah", "masculine": "uh" }
  ],
  "proper_noun_delimiter": ",",
  "conjunctions": {
    "intra_clausal": { "and": "chao", "or": "du" },
    "inter_clausal": {
      "and_also": "zov",
      "and_then": "ze",
      "but": "zuhne",
      "therefore": "zevah",
      "because": "zovo",
      "if_then": "izah",
      "a_if_b": "izo",
      "unless": "izehz"
    }
  }
}
```

### 5.3 idiom-table.json

```json
{
  "idioms": [
    {
      "english_phrase": "up to no good",
      "literal_rephrase": "scheme evil",
      "kryptonese_verb": "regrhahsiv",
      "kryptonese_object": "udol",
      "notes": "Marauder's Map quote — playful mischief"
    },
    {
      "english_phrase": "keep secrets",
      "literal_rephrase": "hide secrets",
      "kryptonese_verb": "throniv",
      "kryptonese_object": "skulev",
      "notes": "keep = hide/conceal in this context"
    }
  ]
}
```

### 5.4 font-mapping.json

```json
{
  "description": "Maps Kryptonese romanization characters to Kryptonian font keyboard characters",
  "mapping": {
    "comment": "To be built by analysing the Kryptonian8.2-Regular.otf character table"
  }
}
```

### 5.5 transliteration-rules.json

```json
{
  "description": "Maps English phonemes to closest Kryptonese equivalents for unknown words",
  "vowel_map": {
    "æ": "a",
    "ɑ": "ah",
    "ɛ": "eh",
    "i": "i",
    "ɪ": "y",
    "o": "o",
    "u": "u",
    "ʌ": "uh",
    "aʊ": "ao",
    "aɪ": "ai",
    "oɪ": "oi"
  },
  "consonant_map": {
    "θ": "th",
    "ð": "dh",
    "ʃ": "sh",
    "ʒ": "zh",
    "ʧ": "ch",
    "ʤ": "j",
    "ŋ": "n",
    "x": "kh"
  }
}
```

---

## 6. Translation Pipeline

### 6.1 English → Kryptonese

```
1. INPUT: Raw English text + context settings
   │
2. SENTENCE SPLIT: Break into individual sentences
   │
3. IDIOM CHECK: Match against idiom-table.json
   │  ├── Match found → replace with literal rephrase
   │  └── No match + unrecognised phrase → flag for user override
   │
4. NLP PARSE (spaCy): For each sentence:
   │  ├── Tokenize
   │  ├── POS tag (verb, noun, adj, adv, etc.)
   │  ├── Dependency parse (subject, object, indirect object)
   │  ├── Detect tense/aspect from verb forms
   │  ├── Detect mood (question from ?, imperative from structure)
   │  └── Detect negation
   │
5. CONTEXT MERGE: Combine auto-detected context with user overrides
   │
6. DICTIONARY LOOKUP: For each content word:
   │  ├── Found → get Kryptonese form + type
   │  └── Not found → phonetic transliteration + flag
   │
7. GRAMMAR ENGINE:
   │  ├── Reorder SVO → VSO
   │  ├── Conjugate verbs (Type-1: stem + TAM suffix, Type-2: select form)
   │  ├── Apply mood prefix if needed
   │  ├── Apply negation prefix (zha-) with correct placement
   │  ├── Handle verb chaining (Type-2 + Type-1)
   │  ├── Insert particles (w, ki, ni)
   │  ├── Process possession (genitive/inalienable/alienable)
   │  ├── Apply gender vowel shifting based on register + context
   │  ├── Apply quantifier suffixes + article harmony
   │  ├── Wrap proper nouns in comma delimiters
   │  ├── Place adverbs between verb and subject
   │  ├── Place postpositions after their noun phrases
   │  └── Join clauses with appropriate conjunctions
   │
8. CONFIDENCE SCORE: Calculate based on:
   │  ├── % words found in dictionary
   │  ├── Number of transliterations
   │  ├── Number of idiom overrides
   │  └── Sentence complexity
   │
9. GLOSS GENERATION: Build word-by-word breakdown table
   │
10. FONT MAPPING: Convert romanization → Kryptonian font characters
   │
11. OUTPUT: Return JSON with:
    ├── romanized Kryptonese text
    ├── Kryptonian font character string
    ├── confidence score
    ├── gloss table
    ├── transliteration notices
    └── any warnings/flags
```

### 6.2 Kryptonese → English

```
1. INPUT: Kryptonese text (romanized or font characters)
   │
2. FONT DECODE: If font characters → map to romanization
   │
3. TOKENIZE: Split into morphemes
   │  ├── Identify mood prefixes
   │  ├── Identify verb stems + TAM suffixes
   │  ├── Identify particles (w, ki, ni, etc.)
   │  ├── Identify proper noun delimiters
   │  └── Identify quantifier suffixes
   │
4. DICTIONARY LOOKUP: Map each morpheme/word to English
   │
5. GRAMMAR REVERSE:
   │  ├── Reorder VSO → SVO
   │  ├── Deconjugate verbs → English tense
   │  ├── Interpret mood prefixes → English modal verbs
   │  ├── Resolve possession structures
   │  └── Reconstruct natural English phrasing
   │
6. OUTPUT: English translation + confidence + gloss
```

---

## 7. API Endpoints

### POST /api/translate

**Request:**
```json
{
  "text": "You don't have to hide secrets anymore, Alex.",
  "direction": "en-to-kryptonese",
  "context": {
    "speaker_gender": "feminine",
    "listener_gender": "feminine",
    "register": "informal",
    "relationship": "family",
    "tense_override": null,
    "mood_override": null
  }
}
```

**Response:**
```json
{
  "romanized": "zhatulem thronivu vahgem rrip w skulevo ,alehks,",
  "glyphs": "...",
  "confidence": "high",
  "gloss": [
    { "word": "zha-tulem", "gloss": "NEG-need.PRS" },
    { "word": "throniv-u", "gloss": "hide-FUT.SIM" },
    { "word": "vahgem", "gloss": "anymore" },
    { "word": "rrip", "gloss": "you.FEM" },
    { "word": "w", "gloss": "COMP" },
    { "word": "skulev-o", "gloss": "secret-PL" },
    { "word": ",alehks,", "gloss": "Alex (proper noun)" }
  ],
  "notices": [],
  "warnings": []
}
```

### POST /api/reverse-translate

Same structure, direction: "kryptonese-to-en"

### GET /api/dictionary

Returns the full dictionary JSON for client-side features (search, browse).

### GET /api/font-mapping

Returns the romanization → font character mapping.

### POST /api/submit-issue

**Request:**
```json
{
  "type": "idiom-submission | translation-feedback",
  "original_text": "up to no good",
  "suggested_rephrase": "scheming evil",
  "translation_output": "...",
  "context": {},
  "user_comment": "From Harry Potter, Marauder's Map"
}
```

Creates a GitHub issue via GitHub API with appropriate label.

---

## 8. Frontend Pages

### 8.1 Home Page (`/`)

```
┌──────────────────────────────────────────┐
│                                          │
│           LINGUA FICTA                   │
│        Latin: "fictional language"       │
│                                          │
│  A free translator for the constructed   │
│  languages of your favourite fictional   │
│  universes. Built by fans, for fans.     │
│                                          │
│  [Start Translating →]                   │
│                                          │
│  ─────────────────────────────────────   │
│                                          │
│  Currently supported:                    │
│  • Kryptonese (Doyle Kryptonian)         │
│                                          │
│  Coming soon:                            │
│  • Rogue (Anna Marie) accent             │
│  • Nightcrawler (Kurt Wagner) accent     │
│  • Star Wars languages                   │
│                                          │
│  ─────────────────────────────────────   │
│                                          │
│  About the Kryptonese language:          │
│  Created by Darren Doyle, a linguist     │
│  who developed a fully-functioning       │
│  language for the Arrowverse.            │
│  → kryptonian.info/doyle                 │
│                                          │
│  ─────────────────────────────────────   │
│                                          │
│  How it works:                           │
│  This translator uses a deterministic    │
│  rules engine — not AI guessing. Every   │
│  translation follows the documented      │
│  grammar and dictionary exactly.         │
│                                          │
│  Words not in the dictionary are         │
│  phonetically transliterated into        │
│  Kryptonese characters, similar to how   │
│  English names are transliterated in     │
│  the show (e.g., Kara → ,kahrah,).      │
│                                          │
└──────────────────────────────────────────┘
```

### 8.2 Translator Page (`/translate`)

```
┌──────────────────────────────────────────┐
│  LINGUA FICTA    [Home] [Translate]      │
│                                          │
│  Language: [Kryptonese (Doyle) ▼]        │
│  Direction: [English → Kryptonese ▼]     │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │ Enter text here...                 │  │
│  │                                    │  │
│  │                                    │  │
│  └────────────────────────────────────┘  │
│                                          │
│  [Translate]                             │
│                                          │
│  ── Context (auto-detected) ──────────   │
│  Speaker: [Neutral ▼]                    │
│  Listener: [Neutral ▼]                   │
│  Register: [Formal ▼]                    │
│  Relationship: [Stranger ▼]              │
│  Tense: [Auto ▼]                         │
│  Mood: [Auto ▼]                          │
│  [✓ Confirm & Translate]                 │
│                                          │
│  ── Output ───────────────────────────   │
│                                          │
│  Kryptonese:                             │
│  zhatulem thronivu vahgem rrip w         │
│  skulevo ,alehks,                        │
│                                          │
│  Kryptonian Script:                      │
│  [rendered in Kryptonian font]           │
│                                          │
│  Confidence: ● High                      │
│                                          │
│  [Show Breakdown ▼]                      │
│                                          │
│  ⚠ "internet" is not in the dictionary. │
│  It has been phonetically transliterated │
│  into Kryptonese characters. This is     │
│  not a native Kryptonese word.           │
│                                          │
│  [Report Issue] [Submit Idiom]           │
│                                          │
└──────────────────────────────────────────┘
```

---

## 9. Deployment (Raspberry Pi 5)

### Stack on Pi
- **OS:** Raspberry Pi OS (64-bit)
- **Web server:** nginx (reverse proxy)
- **Backend:** Python 3.11+ with uvicorn (FastAPI)
- **Frontend:** Static build (React → `npm run build` → served by nginx)
- **Process manager:** systemd service for uvicorn

### nginx config (simplified)
```nginx
server {
    listen 80;
    server_name linguaficta.local;  # or your domain

    location /api/ {
        proxy_pass http://127.0.0.1:8000;
    }

    location / {
        root /var/www/linguaficta/frontend/build;
        try_files $uri /index.html;
    }
}
```

### systemd service
```ini
[Unit]
Description=Lingua Ficta API
After=network.target

[Service]
User=pi
WorkingDirectory=/opt/linguaficta/backend
ExecStart=/opt/linguaficta/backend/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## 10. Repository Structure

```
lingua-ficta/
├── README.md
├── LICENSE
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── idiom-submission.md
│   │   └── translation-feedback.md
│   └── workflows/
│       └── deploy.yml (optional)
│
├── frontend/
│   ├── package.json
│   ├── tsconfig.json
│   ├── public/
│   │   └── fonts/
│   │       └── Kryptonian8.2-Regular.otf
│   ├── src/
│   │   ├── App.tsx
│   │   ├── pages/
│   │   │   ├── Home.tsx
│   │   │   └── Translate.tsx
│   │   ├── components/
│   │   │   ├── InputPanel.tsx
│   │   │   ├── ContextPanel.tsx
│   │   │   ├── OutputPanel.tsx
│   │   │   ├── GlyphRenderer.tsx
│   │   │   ├── BreakdownTable.tsx
│   │   │   ├── ConfidenceIndicator.tsx
│   │   │   ├── IdiomOverrideBox.tsx
│   │   │   ├── FeedbackModal.tsx
│   │   │   └── TransliterationNotice.tsx
│   │   ├── hooks/
│   │   │   └── useTranslate.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── styles/
│   │       └── kryptonian-font.css
│   └── tailwind.config.js
│
├── backend/
│   ├── requirements.txt
│   ├── main.py
│   ├── routers/
│   │   ├── translate.py
│   │   ├── dictionary.py
│   │   └── issues.py
│   ├── engine/
│   │   ├── __init__.py
│   │   ├── parser.py          # English NLP parsing (spaCy)
│   │   ├── rules.py           # Core grammar rules engine
│   │   ├── conjugator.py      # Verb conjugation logic
│   │   ├── possession.py      # Possession system logic
│   │   ├── gender.py          # Gender vowel shifting
│   │   ├── particles.py       # Particle insertion logic
│   │   ├── transliterate.py   # Phonetic transliteration fallback
│   │   ├── idioms.py          # Idiom table lookup
│   │   ├── confidence.py      # Confidence scoring
│   │   ├── gloss.py           # Interlinear gloss generator
│   │   ├── reverse.py         # Kryptonese → English pipeline
│   │   └── font_mapper.py     # Romanization → font chars
│   ├── data/
│   │   ├── dictionary.json
│   │   ├── grammar-rules.json
│   │   ├── idiom-table.json
│   │   ├── font-mapping.json
│   │   └── transliteration-rules.json
│   └── tests/
│       ├── test_translate.py
│       ├── test_conjugator.py
│       ├── test_possession.py
│       ├── test_gender.py
│       └── test_reverse.py
│
├── scripts/
│   ├── parse_dictionary.py    # Convert raw .txt files → dictionary.json
│   └── build_font_mapping.py  # Extract font character table → font-mapping.json
│
└── deploy/
    ├── nginx.conf
    └── linguaficta.service
```

---

## 11. Implementation Order

### Phase 1: Foundation
1. Set up repository structure
2. Parse dictionary .txt files → dictionary.json
3. Build grammar-rules.json from documented grammar
4. Set up FastAPI skeleton with health check endpoint
5. Set up React frontend skeleton with routing

### Phase 2: Core Engine
6. Build English parser (spaCy integration — sentence splitting, POS tagging, dependency parsing)
7. Build verb conjugator (Type-1 TAM suffixes, Type-2 form selection, mood prefixes, negation, verb chaining)
8. Build particle insertion logic (w, ki, ni placement)
9. Build possession system (genitive, inalienable, alienable with fused determiners)
10. Build gender/register vowel shifting
11. Build word order transformer (SVO → VSO)
12. Build quantifier suffix + article harmony
13. Build proper noun transliteration
14. Build phonetic fallback for unknown words
15. Wire all engine components into /api/translate endpoint

### Phase 3: Output & UI
16. Build confidence scorer
17. Build gloss generator
18. Build font character mapping (analyse .otf file)
19. Build frontend translator page — input panel + context panel
20. Build frontend output panel — romanized text + glyph rendering
21. Build breakdown toggle component
22. Build confidence indicator component
23. Build transliteration notice component

### Phase 4: Reverse Translation
24. Build Kryptonese → English morpheme tokenizer
25. Build reverse grammar logic (VSO → SVO, deconjugation)
26. Build font character → romanization decoder
27. Wire into /api/reverse-translate endpoint
28. Add direction toggle to frontend

### Phase 5: Community Features
29. Build idiom table lookup + override box UI
30. Build GitHub issue creation API (idiom submissions)
31. Build feedback modal + GitHub issue creation (translation feedback)
32. Build advisory for multi-speaker detection

### Phase 6: Polish & Deploy
33. Build home page with project info, credits, language explanation
34. Responsive design / mobile support
35. Error handling + edge cases
36. Write tests for all engine components
37. Set up Pi deployment (nginx, systemd, build pipeline)
38. Seed idiom-table.json with common English idioms and their Kryptonese-compatible rephrases

---

## 12. Testing Strategy

### Unit Tests
- Verb conjugation: every tense × aspect combination for sample Type-1 verbs
- Type-2 verb form selection
- Mood prefix application
- Negation placement
- Possession: genitive, inalienable, alienable (with gender variants)
- Gender vowel shifting across all three rows
- Quantifier suffixes + article harmony
- Proper noun transliteration
- Phonetic fallback transliteration

### Integration Tests
- Full sentence translation end-to-end
- Known translations from the site (Lord's Prayer, phrases, proverbs) as regression tests
- The three translations from this conversation as test cases:
  - "You don't have to keep secrets anymore Alex..." → known Kryptonese output
  - "I solemnly swear that I am up to no good." → known output
  - "Mischief managed." → known output

### Edge Cases to Test
- Empty input
- Single word input
- Very long paragraph input
- All punctuation / no words
- All unknown words (pure transliteration)
- Mixed known + unknown words
- Questions (? detection)
- Commands/imperatives
- Multiple sentences with different tenses
- Nested possession
- Verb chains (Type-2 + Type-1)

---

## 13. Future Expansion

### Additional Languages (Phase 7+)
The architecture supports adding new languages by:
1. Creating a new `dictionary-{language}.json`
2. Creating a new `grammar-rules-{language}.json`
3. Creating a new engine module implementing the same interface
4. Adding the language to the frontend selector

### Planned Additions
- **Rogue (Anna Marie)** — Southern accent transformation rules applied to English text
- **Nightcrawler (Kurt Wagner)** — German-accented English transformation rules
- **Star Wars languages** — Mando'a, Huttese, etc. (if sufficient grammar/vocabulary resources exist)

### Architecture for Multi-Language
```
backend/
├── engine/
│   ├── languages/
│   │   ├── kryptonese/
│   │   │   ├── rules.py
│   │   │   ├── conjugator.py
│   │   │   └── ...
│   │   ├── mandoa/
│   │   │   ├── rules.py
│   │   │   └── ...
│   │   └── accents/
│   │       ├── rogue.py
│   │       └── nightcrawler.py
│   └── base.py  # Abstract interface all languages implement
```

---

## 14. Credits & Licensing

- **Language:** Doyle Kryptonian by Darren Doyle — [kryptonian.info/doyle](https://kryptonian.info/doyle/)
- **Font:** Kryptonian8.2-Regular.otf by Darren Doyle — used with permission
- **Project:** Lingua Ficta by Toby Hunter — free and open source
- **License:** TBD (MIT recommended for open fan projects)
