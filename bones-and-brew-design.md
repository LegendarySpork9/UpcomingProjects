# Bones & Brew — Design Document

## Overview

**Bones & Brew** is a cross-platform dice game based on the traditional game of Farkle. Players can compete against AI opponents, locally with friends, or online against other players. The game features a tavern aesthetic, 3D dice animations, a ranked competitive mode, collectible dice, achievements, and a leveling system.

## Name & Branding

- **Name**: Bones & Brew
- **Icon**: A pair of dice next to a tavern tankard
- **Tagline**: TBD

## Platforms

- Windows
- Linux
- iOS
- Android

## Tech Stack

| Component | Technology |
|---|---|
| Game Engine | Godot with C# (.NET 8) |
| Backend API | .NET Web API |
| Database | PostgreSQL |
| Hosting | Raspberry Pi 5 |
| Networking | Routed through Cloudflare |

---

## Core Game Rules

### Standard Farkle Scoring

| Combination | Points |
|---|---|
| Single 1 | 100 |
| Single 5 | 50 |
| Three 1s | 1,000 |
| Three 2s | 200 |
| Three 3s | 300 |
| Three 4s | 400 |
| Three 5s | 500 |
| Three 6s | 600 |
| Four of a kind | 2x the three-of-a-kind value |
| Five of a kind | 4x the three-of-a-kind value |
| Six of a kind | 8x the three-of-a-kind value |
| 1-2-3-4-5-6 straight | 1,500 |
| Three pairs | 750 |

### Gameplay

- Players take turns rolling six dice
- After each roll, the player must set aside at least one scoring die
- The player can then choose to bank their points or roll the remaining dice
- If a roll produces no scoring dice, the player "farkles" and loses all unbanked points for that turn
- **Hot dice**: If all six dice are set aside as scoring dice, the player may re-roll all six and continue their turn
- No minimum opening score required
- The game ends when a player reaches the dice target score (10,000 in ranked). All other players get one final turn to try to beat that score. Highest score wins.

### Ranked vs Custom Rules

| Setting | Ranked | Custom / Local |
|---|---|---|
| Scoring rules | Standard only | Modifiable |
| Dice target score | 10,000 (fixed) | 4,000 — 20,000 (configurable) |
| Dice | Standard only | Up to 3 custom dice allowed |
| Modified rules | Not allowed | Allowed |

---

## Scoring & Match System

### Match Points Formula

At the end of each game, points are calculated:

- **Winner**: Base Points (25) x Multiplier (Y)
- **Y** = Winner's final dice score / 1,000
- **Loser (custom/local)**: 25 points flat
- **Loser (ranked)**: 0 points
- All scores rounded to the nearest whole number

### Examples

| Scenario | Dice Score | Calculation | Match Points |
|---|---|---|---|
| Ranked win | 10,200 | 25 x 10.2 | 255 |
| Ranked win | 14,800 | 25 x 14.8 | 370 |
| Ranked loss | Any | — | 0 |
| Custom win | 10,200 | 25 x 10.2 | 255 |
| Custom loss | Any | Flat | 25 |

### Matches

- A match can consist of multiple games
- Each game plays to the dice target score (10,000 in ranked)
- Match points accumulate across games
- The match ends when a player's cumulative match points reach the match target score
- **Ranked**: Single game (no match target, one game = one ranked result)
- **Custom/Local**: Match target score is configurable (500 — 5,000)

---

## Game Modes

### Single Player vs AI

| Difficulty | Behaviour |
|---|---|
| Easy | Rolls conservatively, banks early |
| Medium | Balanced risk/reward |
| Hard | Optimal play, knows when to push |

- AI opponents have randomised personality traits (aggression, caution) within a range appropriate to their difficulty level
- Hard mode AI can encounter special dice (see Dice Collection)

### Local Multiplayer

- Hot-seat (pass the device)
- 2–6 players, configurable
- Custom rules and dice allowed

### Online Ranked

- 1v1 only
- Standard rules, standard dice
- Matchmaking queue
- 60 seconds of idle time forfeits your turn and any unbanked score

### Online Custom / Private

- 2–6 players, configurable
- Invite codes for private games
- Custom rules and dice allowed
- 60 seconds idle forfeit rule applies

---

## Leaderboard

### Types

| Leaderboard | Scope | Resets | Source |
|---|---|---|---|
| Global | All players | Never | Cumulative ranked match points |
| Seasonal | All players | End of seasonal event | Event-period ranked match points |

### Display

- Top 10 players shown
- Player always sees their own rank and position

### Seasonal Rewards

- Top 3 globally receive a unique badge
- Badge can be equipped to player profile
- Visible to opponents, clickable to see details

---

## XP & Leveling

### XP Rates

| Activity | XP |
|---|---|
| Ranked win | 75 |
| Ranked loss | 25 |
| Custom/Local win | 50 |
| Custom/Local loss | 15 |
| AI Easy win | 25 |
| AI Medium win | 40 |
| AI Hard win | 60 |
| AI loss (any difficulty) | 10 |

### Leveling

- XP required per level = Level x 100 (Level 1 = 100 XP, Level 10 = 1,000 XP, Level 50 = 5,000 XP)
- Rewards every 5 levels (cosmetics, dice, visual flair)
- Reward track is procedural but identical for all players
- Level cap determined by available rewards
- Permanent progression (no seasonal reset)

---

## Achievements

### Categories

- **Score milestones**: Reach X total ranked points (1K, 5K, 10K, 50K)
- **Win milestones**: Win X ranked games (10, 50, 100, 500)
- **Single game feats**: Score 1,500 in one roll (straight), clear the board X times in one turn, win without opponent scoring
- **Streaks**: Win 3/5/10 ranked games in a row
- **Collection**: Beat all AI difficulty levels, collect X dice
- **Seasonal**: Participate in X events, top 3 finish
- **Dice collection**: Collect X special dice, collect all dice from a specific event
- **Personal**: Win your first game, win X matches in private lobbies, win X games against friends

### Achievement Display

- "X% of players achieved this" shown only after the player unlocks the achievement
- All achievements are achievable but some are designed to be genuinely difficult

### Rewards

- Battle pass style: every 5 levels unlocks a reward (visual item, dice, theme element)
- Badges: display player level or seasonal event badge
- Equipped badge visible to opponents, clickable for details

---

## Dice Collection

### Custom Dice Rules

- Maximum of 3 custom dice per non-ranked game
- Duplicates of the same die are allowed
- Custom dice cannot be used in ranked games

### Special AI Dice Encounters (Hard Mode Only)

**Encounter chance**: 16.67% per hard mode game (1 in 6 games)

The AI carries one special die per encounter. The player can see the die is different during gameplay, but details are only revealed in the dice catalog upon winning it.

### Tier Distribution (Given Encounter)

| Tier | Chance | Overall per Game | Approx. Games to Get |
|---|---|---|---|
| Common | 55% | 9.17% | ~11 |
| Uncommon | 30% | 5.0% | ~20 |
| Rare | 12% | 2.0% | ~50 |
| Legendary | 3% | 0.5% | ~200 |

### Dice Properties by Tier

- **Common**: Slightly different visuals, minor flavour
- **Uncommon**: Noticeable cosmetic differences, small weighted edge
- **Rare**: Strong visual flair, meaningful weight advantage
- **Legendary**: Unique named dice, best stats, highly distinctive visuals

### Seasonal Event Dice

- During seasonal events (Easter, Halloween, Christmas), the app theme changes
- Special seasonal AI opponents have a chance of appearing with event-exclusive dice
- Event dice follow the same tier and encounter system
- Players must beat the AI to obtain the dice

### Dice Catalog

- Accessible from the main menu
- Browse full collection, equip dice, preview appearance
- Shows dice details for any dice the player has won

---

## Seasonal Events

- Tied to real-world holidays (Easter, Halloween, Christmas)
- App theme changes during the event period
- Seasonal AI encounters are randomised (not guaranteed every game)
- Seasonal leaderboard runs for the duration of the event
- Top 3 on the seasonal leaderboard receive a unique badge
- Event-exclusive dice available only during the event period

---

## UI / UX

### Art Style

- Tavern aesthetic throughout
- Themed variations during seasonal events
- Host's custom theme applies to all players in their lobby

### Game Board

- Player's dice (interactive — tap/click to keep, tap/click again to release)
- Opponent's dice (visible, non-interactive)
- Both players' total scores
- Turn indicator
- Bank button
- 3D physics-based dice roll animation

### Audio

- Dice clatter sound effects on roll
- Theme-appropriate ambient music and tavern sounds
- Audio themes change with seasonal events

### Screen Flow

```
Main Menu
├── Play
│   ├── vs AI → Difficulty Select → Game → Results → AI Menu
│   ├── Local Multiplayer → Lobby Setup → Game → Results → Lobby
│   ├── Ranked → Matchmaking → Game → Results → Matchmaking
│   └── Custom Online → Lobby (invite code) → Game → Results → Lobby
├── Dice Catalog
├── Achievements
├── Leaderboard
├── Profile / Settings
└── Account
```

### Platform Adaptation

- Same game on all platforms
- Layout adapted for mobile screens (details determined during design/prototyping)

---

## Persistence & Accounts

### Accounts

- Required only for online play, ranked, and leaderboard
- Not required for offline AI or local multiplayer
- Auth: Username and password with authenticator-based 2FA

### Player Profile

- Display name
- Equipped badge (player level or seasonal event badge)
- Equipped dice (up to 3 for custom games)
- Equipped theme

### Data Sync

- All progression stored server-side
- Local cache on device for offline play
- On login: server syncs full profile to local device
- On reconnect: local changes merge back to server
- Conflict resolution: server timestamp wins, offline progress merges in (does not overwrite)

### Offline Play

- AI games playable without an internet connection
- Offline games earn XP and achievements (not ranked points or leaderboard score)
- Progress syncs to server on reconnect after anti-cheat validation

---

## Server Connection

### Startup Check

- On app launch, the client performs a health check against the backend API
- If the server is unreachable or returns an error, online-dependent features are disabled in the UI

### Feature Availability by Connection Status

| Feature | Server Available | Server Unavailable |
|---|---|---|
| vs AI | Available | Available |
| Local Multiplayer | Available | Available |
| Ranked Matchmaking | Available | Greyed out / disabled |
| Custom Online Lobbies | Available | Greyed out / disabled |
| Leaderboard | Available | Greyed out / disabled |
| Achievements | Available (synced) | Available (local cache) |
| Dice Catalog | Available | Available (local cache) |
| Profile / Account | Available | View only (local cache) |

### Behaviour

- Disabled menu options display a message explaining the server is unavailable (e.g. "Server offline — online features unavailable")
- The client periodically retries the connection in the background while the app is open
- When the connection is restored, online features are re-enabled automatically without requiring an app restart
- If the connection drops mid-game during an online match, the player is given a grace period to reconnect before the match is forfeited

---

## Anti-Cheat (Offline Games)

Offline games that earn XP and achievements are validated on reconnect:

- **Game replay log**: Every roll result, dice selection, and bank decision stored locally
- **Signature/checksum**: Game client signs each action with a session-derived key to prevent casual save file editing
- **Sanity checks**: Server flags impossibilities (too many games in too short a time, impossible roll patterns, XP that doesn't match the game log)
- **Flagging not blocking**: Suspicious activity is flagged for review rather than instantly rejected to avoid punishing legitimate players

---

## Performance Scaling

The game is turn-based with lightweight API calls (score submissions, matchmaking, leaderboard queries, sync). This means the server load per player is low compared to real-time action games. The bottlenecks are concurrent connections, database writes, and matchmaking during peak hours.

### Assumptions

- ~10% of total players are online concurrently at peak
- Each online game generates ~20–40 API calls (auth, matchmaking, turn syncs, score submission, leaderboard update)
- A ranked game lasts ~5–10 minutes on average
- Leaderboard reads are the most frequent query; score writes are bursty at game end

### Scaling Tiers

| Tier | Total Players | Peak Concurrent | Host | Estimated Cost |
|---|---|---|---|---|
| 1 | Up to 500 | ~50 | Raspberry Pi 5 (8GB) | Existing hardware |
| 2 | 500 — 5,000 | ~500 | Small VPS (2 vCPU, 4GB RAM) or dedicated mini PC | ~£10–20/month |
| 3 | 5,000 — 50,000 | ~5,000 | Mid-range VPS (4 vCPU, 8GB RAM) with managed PostgreSQL | ~£40–80/month |
| 4 | 50,000 — 250,000 | ~25,000 | Dedicated server (8 vCPU, 16GB+ RAM) with connection pooling and read replicas | ~£100–200/month |
| 5 | 250,000+ | 25,000+ | Cloud infrastructure (auto-scaling, load balanced, managed DB with replicas) | Variable, scales with usage |

### Pi 5 Limitations

- **CPU**: Quad-core Cortex-A76 @ 2.4GHz handles the .NET API well at low concurrency but will struggle beyond ~50–100 concurrent connections
- **RAM**: 8GB shared between the .NET API and PostgreSQL leaves limited headroom under load
- **Storage**: SD card or USB SSD — database I/O becomes a bottleneck as the player table and replay logs grow
- **Network**: Single gigabit ethernet through Cloudflare — adequate for low traffic, but Cloudflare rate limits and tunnel throughput may cap before hardware does
- **PostgreSQL**: Connection limit and write throughput on the Pi will be the first constraint hit

### Warning Signs to Monitor

| Metric | Healthy | Time to Upgrade |
|---|---|---|
| API response time (p95) | < 200ms | > 500ms sustained |
| Database connection pool usage | < 70% | > 85% sustained |
| CPU usage (sustained) | < 60% | > 80% for 10+ minutes |
| RAM usage | < 70% | > 85% |
| Matchmaking wait time | < 10 seconds | > 30 seconds sustained |
| Failed connection rate | < 1% | > 5% |
| Database disk usage | < 70% capacity | > 80% capacity |

### Scaling Strategy

1. **Start on Pi 5** — Sufficient for launch and early growth. Monitor the warning signs above.
2. **Optimise before migrating** — When metrics approach thresholds, first apply optimisations:
   - Add database connection pooling (e.g. PgBouncer)
   - Cache leaderboard queries (refresh every 30–60 seconds rather than live)
   - Batch offline sync validations rather than processing immediately
   - Add database indexes as query patterns become clear
3. **Migrate to VPS** — When optimisations are exhausted, move to a VPS. The .NET API and PostgreSQL are portable with minimal changes.
4. **Separate API and database** — At Tier 3+, run the API and database on separate hosts to scale independently.
5. **Add read replicas** — Leaderboard and achievement stats queries can hit a read replica to reduce load on the primary database.
6. **Load balancing** — At Tier 5, run multiple API instances behind a load balancer. The API should be stateless (all state in the database) to support this from day one.

### Revenue vs Cost Reference

At £2.50 per sale:

| Players | Revenue | Approx. Monthly Hosting |
|---|---|---|
| 500 | £1,250 | £0 (Pi 5) |
| 5,000 | £12,500 | ~£20 |
| 50,000 | £125,000 | ~£80 |
| 250,000 | £625,000 | ~£200 |

Hosting costs remain a negligible percentage of revenue at every tier.

### Design Considerations for Scaling

- **Stateless API**: No session state stored in the API process. All state lives in the database or client cache. This ensures any API instance can handle any request, enabling horizontal scaling later.
- **Database migrations**: Use versioned migrations from the start so schema changes deploy cleanly across environments.
- **Cloudflare caching**: Static assets and leaderboard snapshots can be cached at the Cloudflare edge to reduce load on the origin server.
- **Replay log retention**: Define a retention policy for offline game replay logs (e.g. delete after validation + 30 days) to prevent unbounded database growth.

---

## Donations & Running Costs Transparency

### Donation Prompt

A non-intrusive pop-up appears periodically (e.g. once per month, or after a set number of online games) inviting the player to contribute toward server running costs. The prompt should:

- Be dismissible with a single tap/click and never block gameplay
- Not reappear for the configured interval after dismissal
- Never appear during a game — only on the main menu or results screen
- Be skippable permanently via a "don't show again" toggle in settings

### Donation Page

Accessible at any time from the main menu (e.g. a "Support the Game" option). The page includes:

- **Current monthly running cost**: A plain-language breakdown of what the server costs to run (e.g. "This month, hosting costs £20 to keep online services running for X players")
- **Donation options**: A few preset amounts (e.g. £1, £2.50, £5, £10) and a custom amount field
- **Payment method**: Integrated payment provider (e.g. Stripe, PayPal)

### Disclaimer

Displayed prominently on the donation page:

> All donations go directly toward the running costs of online services (hosting, database, networking). Any surplus is held as a buffer fund up to a maximum threshold of [£X]. Once the buffer is full, all additional donations are forwarded to [chosen charity]. Donations are voluntary and do not grant any in-game advantage.

### Fund Allocation

| Priority | Destination | Details |
|---|---|---|
| 1 | Running costs | Monthly hosting, database, networking, domain, and Cloudflare costs |
| 2 | Buffer fund | Surplus saved as a safety net, capped at a defined threshold (e.g. 6 months of running costs at the current tier) |
| 3 | Charity | Once the buffer is full, all further surplus is donated to a chosen charity |

### Configurable Values (To Be Decided)

- Buffer fund threshold (suggested: 6 months of current running costs)
- Charity recipient
- Donation prompt frequency (suggested: once per month)
- Whether to display a public running costs dashboard (e.g. "This month's costs: £X / Donations received: £Y / Buffer: £Z")

### Transparency Dashboard (Optional)

A page accessible from the main menu or donation page showing:

- Current month's running costs
- Total donations received this month
- Current buffer fund balance and capacity
- Total donated to charity to date

This builds trust and encourages continued support by showing players exactly where their money goes.

---

## Legal Considerations

### Game Name & Intellectual Property

- **Farkle trademark**: The name "Farkle" is trademarked by PlayMonster for their specific product. To avoid any trademark issues, the game should be released under an original name. The underlying game mechanics are traditional and cannot be copyrighted.
- **Kingdom Come: Deliverance**: The game is inspired by KCD2's dice mini-game, but the mechanics are traditional Farkle. Do not use any KCD2 branding, artwork, dice designs, or marketing that implies affiliation with Warhorse Studios or Deep Silver.
- **Original assets**: All art, audio, and UI assets must be original or properly licensed.

### Terms of Service

A Terms of Service agreement is required and must cover:

- Account creation and eligibility (minimum age)
- Acceptable use and player conduct in online games
- Account suspension and termination rights
- Intellectual property ownership (the game and its content)
- Limitation of liability
- Donation terms (voluntary, no in-game advantage, non-refundable)
- Service availability (no guarantee of uptime)
- Modification of terms (right to update with notice)

### Privacy Policy

A Privacy Policy is required by law (UK GDPR) and by app store policies. It must cover:

- What personal data is collected (username, email, password hash, IP address, gameplay data)
- Why it is collected (account management, matchmaking, leaderboards, anti-cheat)
- How it is stored and protected (encrypted, server location)
- Data retention periods
- Third-party data sharing (payment processors, analytics if any)
- User rights under UK GDPR: access, rectification, deletion, portability, objection
- Cookie and tracking disclosure (if applicable)
- Contact details for data-related requests
- How minors' data is handled

### UK GDPR / Data Protection Act 2018

- Lawful basis for processing must be established (likely legitimate interest for gameplay, consent for marketing if any)
- Data Protection Impact Assessment may be needed given the scale of data processing
- Right to deletion: players must be able to request full account and data deletion
- Data breach notification procedures (72-hour reporting requirement to the ICO)
- Consider whether registration with the ICO is required (likely yes if processing personal data commercially)

### Age Rating

- App stores require age classification (PEGI in Europe, ESRB in North America)
- A dice game with no real-money gambling is likely rated PEGI 3 or PEGI 7
- The random dice reward system (special AI encounters) should be reviewed against loot box guidelines — while no real money is spent to obtain them, some jurisdictions are tightening rules around random reward mechanics
- If donations are accepted in-app, this may affect the age rating or require parental consent flows

### App Store Compliance

| Platform | Revenue Cut | Small Business Rate | Notes |
|---|---|---|---|
| Apple App Store | 30% | 15% (under $1M/year revenue) | Privacy policy required. In-app donation must follow Apple's payment rules. |
| Google Play Store | 30% | 15% (first $1M/year) | Privacy policy required. Data safety section must be completed. |
| Steam (if applicable) | 30% | 25% (after $10M) / 20% (after $50M) | No small business programme but no donation restrictions. |

At a £2.50 sale price under the small business programmes:

| Platform | Apple Takes | Google Takes | Developer Receives |
|---|---|---|---|
| Apple | ~£0.38 | — | ~£2.12 |
| Google | — | ~£0.38 | ~£2.12 |

### Donations & Charity

- **Collecting donations**: Accepting voluntary donations for running costs is generally permissible, but the terms must be clear (not a purchase, no goods or services in return, non-refundable)
- **Forwarding to charity**: Collecting money with the stated intent of donating surplus to charity may trigger regulations depending on jurisdiction. Options:
  - Partner formally with a registered charity
  - Use a platform like JustGiving or GoFundMe for the charity portion
  - Register as a charity fundraiser with the relevant authority (Charity Commission in England & Wales)
- **Tax implications**: Donation income that covers running costs is likely taxable as business income. Charitable donations made by the business may be tax-deductible. Consult an accountant.
- **Transparency obligations**: If publicly stating that funds go to charity, there is a legal and ethical obligation to follow through. The transparency dashboard helps demonstrate this.

### Gambling Classification

- The game involves dice but no real-money wagering — this is not classified as gambling
- Custom dice are earned through gameplay, not purchased — no loot box purchase mechanism
- However, the random reward system (special AI dice encounters) should be monitored against evolving regulations, particularly in Belgium, the Netherlands, and under any future UK legislation on random reward mechanics in games
- As long as custom dice cannot be traded, sold, or converted to real-world value, the risk is low

### Recommended Actions Before Launch

1. Choose an original game name and check trademark availability
2. Commission or generate a Terms of Service and Privacy Policy (services like Termly or iubenda, or consult a solicitor)
3. Register with the ICO as a data controller if required
4. Consult a solicitor on the charity donation forwarding mechanism
5. Consult an accountant on tax treatment of sales, donations, and charitable contributions
6. Complete age rating classification (PEGI / IARC)
7. Review Apple and Google's in-app donation and payment policies for compliance
8. Ensure GDPR-compliant consent flows are built into account creation

---

## Development Phases

### Phase 1 — Core Game

- Single player vs AI (Easy / Medium / Hard with personality variation)
- Core Farkle gameplay with standard scoring
- 3D dice roll animation
- Tavern UI theme with dice clatter SFX and ambient audio
- Account creation and login with 2FA
- Basic leveling and XP system
- Server-side sync with local cache
- Windows and Linux builds

### Phase 2 — Online & Progression

- Online ranked 1v1 with matchmaking
- Global and seasonal leaderboards
- Achievements system
- Dice collection and catalog
- Offline anti-cheat validation
- Match system (multi-game matches with match target scores)

### Phase 3 — Expansion

- iOS and Android builds
- Custom / private lobbies with invite codes
- Local multiplayer (hot-seat, 2–6 players)
- Seasonal events with themed UI and special AI dice encounters
- Themes and cosmetic rewards
- Battle pass reward track (every 5 levels)
- Spectating (post-MVP consideration)
