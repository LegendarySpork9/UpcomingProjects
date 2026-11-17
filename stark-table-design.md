# Stark Table — Design Document

## Overview

**Stark Table** is a custom-built smart table with a hidden motorised touchscreen workstation. When closed, it looks like a normal wooden coffee table. A three-stage hidden activation sequence opens the table — four surface quadrants slide outward in both dimensions, a monitor rises flush into the central gap, and a fingerprint sensor authenticates the user. The touchscreen runs a custom sci-fi themed UI for drawing/sketching, mapping, diagramming, and 3D model design.

## Inspiration

Tony Stark's holographic workbench from the Iron Man films.

---

## Physical Build

### Table Dimensions

| Property | Specification |
|---|---|
| Footprint (closed) | ~90-100cm x 55-65cm |
| Footprint (open) | ~105-115cm x 70-80cm |
| Height | Hip height (~90-100cm), motorised adjustable |
| Weight estimate | 40-60kg total |
| Surface use | Light duty — drinks, books, phone |

### Materials

| Component | Material |
|---|---|
| Surface | Walnut veneer on birch plywood (20-25mm thick) |
| Edge profile | Slight chamfer |
| Finish | Matte lacquer + ceramic coating (spill + wear protection) |
| Seam hiding | Shadow line grooves routed into the surface — real split lines hidden among decorative grooves |
| Skirt | Removable magnetic panels with 20-30mm gap at bottom for ventilation |

### Opening Mechanism

The table surface splits into **four quadrants**. All four slide outward simultaneously — the two side pieces move left/right and the two end pieces move front/back — expanding the table in both dimensions to reveal a central rectangular gap. The monitor rises from below into this gap.

| Component | Specification |
|---|---|
| Drive method | Linear actuators (12/24V DC) — one per quadrant, one for monitor lift |
| Total actuators | 5 (4 slides + 1 lift) |
| Speed | ~6-8 seconds full open + monitor raised (tuneable) |
| Locking (closed) | Self-locking actuators + magnetic alignment pins for seam precision |
| Locking (open) | Self-locking actuators hold position |
| Manual release | Quick-disconnect pin/thumb screw on each actuator mount for disassembly |
| Monitor lift | Scissor lift with linear actuator, ~30-50mm travel |
| Monitor tilt | Optional — independently controlled front/rear lift actuators for 5-15 degree tilt |

### Height Adjustment

| Property | Specification |
|---|---|
| Mechanism | Commercial motorised standing desk frame |
| Control | Separate hidden control from the table open/close mechanism |
| Table top and housing | Built onto the motorised frame |

---

## Activation Sequence

### Opening (three-stage)

1. **Magnetic reed switch** — Hold a specific magnet (disguised as coaster/keyring) near a hidden spot on the table. Arms the system. 30-second timeout if step 2 isn't completed.
2. **Capacitive touch sensor** — Tap a hidden zone on the table surface. Triggers the table opening — four quadrants slide apart, monitor rises.
3. **Fingerprint sensor** — Touch the hidden fingerprint reader (table edge). Wakes and unlocks the monitor. Screen stays black until authenticated.

### Closing

1. **Long-press** the capacitive touch sensor — monitor locks/sleeps, monitor lowers, four quadrants slide back together, system locks (same as timeout lock).

### Safety

| Feature | Implementation |
|---|---|
| Primary | IR/optical beam-break sensors along inner edges of all four quadrants and the monitor bay. Movement stops and reverses if beam is broken. |
| Secondary | Current-limiting on all actuators — stops and reverses on unexpected resistance |
| Passive | Slow speed (~6-8 seconds) gives reaction time |
| Closing protection | Sensors detect fingers or objects in gaps, prevent closing if obstructed |

---

## Power Failure Behaviour

| Scenario | Behaviour |
|---|---|
| Power cuts while open/mid-travel | Self-locking actuators hold position — table stays where it is |
| Power restored | Auto-close sequence — table returns to hidden/closed state |
| Battery backup | Not needed — actuators hold mechanically without power |

---

## Display

### Touchscreen

| Property | Specification |
|---|---|
| Model | Huion Kamvas Pro 27 (144Hz) |
| Size | 26.9" (27") |
| Resolution | 3840 x 2160 (4K) |
| Refresh rate | 144Hz |
| Panel type | IPS, fully laminated, anti-glare |
| Touch | PCAP 10-finger multi-touch |
| Pen | Huion PenTech 4.0, EMR (battery-free), 16,384 pressure levels, 60-degree tilt |
| Dimensions | 656 x 405 x 22.7mm |
| Weight | 7.2kg |
| Cooling | Passive (no fans) |
| Colour | 99% sRGB, 98% Adobe RGB, 98% DCI-P3 |
| Orientation | Landscape, flat/horizontal, fixed (with optional tilt) |
| Price | ~£1,800 |

### Screen Protection

| Feature | Implementation |
|---|---|
| Spill protection | Silicone gasket around monitor perimeter in the flush-mount gap |
| Scratch protection | Replaceable matte screen protector film (~£10-15 per replacement) |
| Table surface | Hydrophobic ceramic coating over matte lacquer (reapply every 2-5 years) |

---

## Computer

### Mini PC

| Property | Specification |
|---|---|
| Model | Minisforum 795S7 |
| CPU | AMD Ryzen 9 7945HX (Zen 4, 16-core, 5.4GHz) |
| GPU | NVIDIA RTX 4060 Desktop (low-profile, PCIe 5.0 x16 slot) |
| RAM | 32GB DDR5 |
| Storage | 1TB NVMe SSD |
| Dimensions | 300 x 225 x 95mm |
| Weight | ~2kg |
| VRAM | 8GB GDDR6 |
| OS | Linux (Ubuntu minimal or Fedora) |
| Mounting | Frame crossbar via VESA bracket with rubber vibration-dampening grommets |
| Price | ~£700 |

### GPU Upgrade Path

The 795S7's low-profile PCIe slot allows GPU upgrades within the low-profile card selection. If a larger GPU is ever needed, the mounting design accommodates a switch to a Beelink GTi14 Ultra + EX Dock (~£1,350) or similar eGPU setup — just alter the mounting position.

### Noise Mitigation

| Measure | Detail |
|---|---|
| Vibration damping | Rubber grommets between VESA bracket and frame (prevents wood resonance) |
| Fan curve | Tuned in BIOS to prioritise quiet over max performance |
| Acoustic foam | Optional pad on table underside above PC |
| Mounting position | On frame crossbar, not table surface (metal doesn't resonate like wood) |
| Future option | Noctua NF-A4x10 (40mm) fan swap for CPU cooler, or aftermarket LP GPU cooler |

### Thermal Management

| Requirement | Solution |
|---|---|
| Airflow | Mini PC mounted in open air on frame — not enclosed in a cavity |
| Ventilation | Skirt panels have 20-30mm gap at bottom, removable magnetic panels for maintenance |
| Heat sources | Mini PC is the only significant heat source; monitor is passively cooled (60W) |

---

## Audio

| Property | Specification |
|---|---|
| Type | Surface transducers (exciters) — bolted to underside of table surface |
| Effect | Table top itself vibrates to produce sound — completely invisible |
| Amplifier | Small amp board (12V DC) |
| Use cases | UI sound effects, virtual assistant voice output, ambient sci-fi sounds |
| Cost | ~£50-70 (transducers + amp) |
| Optional upgrade | Hidden subwoofer mounted in frame for bass if ever needed |

---

## Lighting

### LED System

| Zone | Description |
|---|---|
| Gap lighting | LED strips along inner edges of four quadrants — light spills from opening gaps as table splits |
| Monitor bay underglow | LEDs in monitor cavity — glows upward as monitor rises |
| Under-table glow | LED strip along bottom of skirt — table appears to float when active |

### Behaviour

| Property | Detail |
|---|---|
| LED type | WS2812B addressable strips (individually controlled colour, brightness, animation) |
| Theme-linked | LED colour matches active UI theme (Stark = blue, Tron = cyan, Cyberpunk = pink, etc.) |
| Animation | Fade in during open, gentle pulse while active, fade out during close |
| Controller | Dedicated LED controller driven by main controller board |
| Cost | ~£30-50 |

---

## Cable Routing & Power

### Cable Routing

| Property | Detail |
|---|---|
| Design | Fully self-contained — all components internal to the table |
| External cables | Single mains power cable exits through a table leg |
| Network | Wi-Fi from mini PC (or optional ethernet run alongside power cable) |
| Monitor cables | Coiled/retractable cables with slack loop to handle monitor movement |
| USB access | Single USB extension routed to hidden flush-mount port on table edge (magnetic cover) |

### Power Distribution

| Component | Power |
|---|---|
| Mini PC | 240V AC (own power brick) |
| Huion Kamvas Pro 27 | 240V AC (own power adapter) |
| Motorised desk frame | 240V AC (own control box) |
| Linear actuators x5 | 12/24V DC |
| Controller board | 12V DC |
| Surface transducers + amp | 12V DC |
| IR sensors, reed switch, capacitive sensor, fingerprint | 5V DC |
| LED strips + controller | 12V DC |

| Distribution | Detail |
|---|---|
| Mains entry | Single cable through table leg |
| AC distribution | Hidden multi-socket strip for three 240V components |
| DC power supply | Single 24V PSU with step-down converters to 12V and 5V |
| Protection | Inline fuse on mains input; DC PSU handles overcurrent |
| Wiring | Bundled and labelled for maintenance traceability |

---

## Maintenance & Access

| Access Point | Method |
|---|---|
| Mini PC | Remove relevant magnetic skirt panel — PC is on frame crossbar |
| Actuators | Accessible from below via skirt panel removal + manual release pins |
| Controller board & power | Centralised electronics bay on frame — single skirt panel removal |
| Monitor & scissor lift | Accessible from below when table is open; manual release on slides for closed access |
| LED strips & sensors | JST plug connectors at junction points — unplug and replace, no soldering |
| General principle | Everything uses plug connectors, every component removable independently |

---

## Software

### Architecture

| Property | Detail |
|---|---|
| Framework | Electron app with React frontend |
| Rendering | Three.js (WebGL, with WebGPU path for future) |
| Boot behaviour | Linux boots straight into Electron app fullscreen — no desktop visible |
| Feel | "This is a device, not a PC" |

### UI Themes

The entire UI is driven by a theme engine. All colours, glows, animations, fonts, and UI shapes are defined in theme files. Switching themes swaps the entire look instantly.

| Theme | Aesthetic |
|---|---|
| **Stark** (default) | Translucent blue panels, floating HUD elements, thin glowing lines, circular arc menus |
| **Tron** | Neon lines on black, geometric, minimal |
| **Cyberpunk** | Neon pink/cyan, glitchy, layered, busy |
| **Imperial** | Clean, angular, black/grey/red, structured |
| **The Expanse** | Dark, utilitarian, dense information, amber/orange |
| **Mass Effect** | Orange holographic, clean geometric |

| Behaviour | Detail |
|---|---|
| Default theme | Stark |
| Persistence | Last selected theme saved to local storage, persists between restarts |
| LED sync | LED colours change to match active theme |
| Nav element | Themed per-theme (blue arc in Stark, neon lines in Tron, etc.) |
| Switching | Via UI toggle or voice command to assistant ("switch to Tron mode") |

### Hub / Launcher

| Property | Detail |
|---|---|
| Appears | After fingerprint authentication |
| Layout | Central dashboard with mode tiles — Drawing, Maps, Diagrams, 3D Viewer, Settings |
| Transitions | Hub morphs/animates into the selected mode workspace (no separate windows) |
| Navigation | Persistent themed nav element (edge swipe or floating arc menu) to switch modes or return to hub |
| Assistant | Accessible from anywhere — voice activation or persistent UI element in every mode |
| Fully themed | Every element follows the active theme |

### Wake / Sleep Behaviour

| State | Trigger | Action |
|---|---|---|
| Active | User interaction | Sci-fi hub or active mode displayed |
| Screensaver | 5 minutes idle | Themed animated ambient display (Stark = rotating holographic wireframes, Tron = light cycles, etc.) Touch wakes instantly |
| Dimmed | 15 minutes idle | Screen brightness drops to 10% |
| Auto-close | 30 minutes idle | Full shutdown sequence — monitor sleeps, lowers, table closes, system locks |
| All timings | Configurable in settings | |

---

## Drawing / Sketching

| Property | Detail |
|---|---|
| Types | Both raster (pixel-based) and vector (scalable) |
| Pressure sensitivity | Full support — software responds to pen pressure and tilt |
| Canvas library | HTML Canvas / WebGL (fully custom, skinnable with theme) |
| Pen support | Huion EMR — 16,384 pressure levels, 60-degree tilt |
| Touch support | PCAP 10-finger multi-touch for gestures |

---

## Mapping

A unified mapping hub integrated into the sci-fi UI, with multiple sources accessible via the UI or voice commands to the assistant.

| Source | Use Case |
|---|---|
| Google Maps | Routing, street view, traffic, directions |
| Google Earth | 3D terrain, satellite imagery, flyovers |
| OpenStreetMap | Open source, offline-capable |
| Bing Maps/Satellite | Alternative satellite imagery |
| OS Maps | Ordnance Survey UK terrain/walking detail |
| Mapbox | Custom embedded maps, skinnable for sci-fi UI |

| Feature | Detail |
|---|---|
| Switching | Assistant handles source switching ("show me satellite of my house" → Google Earth) |
| Offline | OpenStreetMap provides offline capability |
| Integration | APIs embedded in the custom UI via Mapbox/Leaflet |

---

## Diagramming

| Property | Detail |
|---|---|
| Types | Flowcharts, network diagrams, mind maps, floor plans, circuit/schematics, UML, furniture/product design |
| Freehand mode | Draw with pen, stays as-is |
| Shape recognition | Draw rough shapes, snap to clean geometric shapes |
| Structured tools | Drag and drop shapes, connectors, text labels from a palette |
| Mixed mode | Freehand annotations on top of structured diagrams |
| Library | Fabric.js or Konva.js (skinnable with theme) |

---

## Sketch-to-3D Pipeline

### Conversion Method — Hybrid (Parametric + AI)

| Stage | Method | Detail |
|---|---|---|
| Initial creation | Parametric (user-driven) | Draw 2D views (front, side, top), label dimensions. Software extrudes and combines into 3D based on defined measurements. Lightweight CAD. |
| Iteration | AI/Assistant-driven | "Add extra shelving," "change wood to walnut," "make top shelf narrower." Assistant modifies the parametric model intelligently. |
| Manual tweaks | Touch/pen direct | Drag a shelf, change a dimension, adjust directly on the 3D model. |

### Bidirectional 2D-3D Link

| Feature | Detail |
|---|---|
| Live link | 2D sketches and 3D model are views of the same underlying data |
| 2D → 3D | Edit a 2D view, 3D model updates |
| 3D → 2D | Edit in 3D (via assistant or manual), 2D views reflect the change |
| History | Full undo/redo, step back and forth between any stage |
| Navigate | Switch freely between 2D sketch views and 3D model at any time |

### 3D Detail Levels

| Mode | Detail | Use Case |
|---|---|---|
| **Standard (default)** | Bevels, rounded edges, chamfers, routed profiles — actual shape-affecting details | Design evaluation — see how edge profiles change the feel |
| **Construction detail (toggle)** | Grain direction, dovetail joints, screw holes, hardware fittings | Build planning — ready to take to the workshop |

### Materials & Colour

| Property | Detail |
|---|---|
| Rendering | Real PBR materials — texture, roughness, reflections, bump/normal mapping |
| Assignment | Per-face / per-region — any surface or defined area can have its own material independently |
| Library | Woods (walnut, oak, ash, pine, birch ply, MDF), finishes (raw, oiled, lacquered, painted with colour picker), metals (steel, brass, black iron), fabrics/leather |
| Expandable | Import custom materials — photograph a real wood sample, add as texture |
| Assistant integration | "Change the shelves to ash," "make all the circles blue," "show painted white" |
| Multi-dimensional | Different materials on any defined region of any part (e.g., box faces red, circles on faces blue) |

### 3D Viewer

| Property | Detail |
|---|---|
| Engine | Three.js (WebGL, WebGPU future path) |
| Rotation | Full 360 on all axes — spin, tilt, flip |
| Zoom | Pinch to zoom on touchscreen |
| Rotate control | Pen drag to rotate |
| Rendering | Real-time PBR at 4K |

### Lighting

**Default mode (~60-90fps):**

| Technique | Frame Cost | VRAM |
|---|---|---|
| 4K HDR environment (IBL) | 2-3ms | ~16MB |
| Studio lights x3, PCF8, 2048² shadows on key light only | 6-8ms | ~48MB |
| Ambient fill | 0.1ms | 0 |
| SSAO at half-resolution, 32 samples | 2-3ms | ~50MB |
| TAA anti-aliasing | 1-2ms | — |
| **Total** | **~11-16ms** | **~114MB** |

**Quality mode (~30fps, for presentation renders):**

| Additional Technique | Frame Cost | VRAM |
|---|---|---|
| SSR on glossy surfaces (half-res) | 5-8ms | ~128MB |
| Full-resolution SSAO, 64 samples | 12-16ms | ~200MB |
| 5 light probes for indirect colour | 1ms | Negligible |
| 8K HDR environment | 3-4ms | ~64MB |
| All 3 lights casting shadows at PCF16 | 9-12ms | ~48MB |

**Additional lighting features:**

| Feature | Detail |
|---|---|
| Custom light positioning | Drag lights around the scene, adjust colour temperature (2700K-6500K) and intensity |
| Time of day | Procedural sky, sun position slider, shadow direction and colour shifts |
| Performance/Quality toggle | UI switch with warning on quality mode for complex models |

**Not available (browser limitation):**

| Feature | Reason |
|---|---|
| Hardware ray tracing | WebGL has no RT API. WebGPU RT spec estimated 2027+. RTX 4060 RT cores unused in browser. |

**VRAM budget:** Default mode ~1.9GB of 8GB. Quality mode ~3.5-4GB. Ample headroom.

---

## Export

### File Formats

| Format | Use Case |
|---|---|
| STL / 3MF | 3D printing |
| OBJ / FBX | Import into Blender, SketchUp, Fusion 360, etc. |
| PDF with dimensions | Traditional 2D plans for the workshop |
| PNG / JPEG renders | Snapshots from specific angles for sharing |
| DXF | CNC cutting / laser cutting |

### 3D Print Auto-Split

| Feature | Detail |
|---|---|
| Auto-split | Models too large for a print bed are automatically split into printable sections |
| Connectors | Pin and hole connectors added at each split point for alignment |
| Override | User can choose split direction and locations manually |
| Output | Each piece as a separate labelled and numbered STL/3MF file |

### Export Methods

| Method | Detail |
|---|---|
| Network share | Export to shared folder on home network |
| Cloud upload | Google Drive, OneDrive, Dropbox |
| Email | Direct from the UI |
| Bluetooth / AirDrop | Push to nearby phone or laptop |
| Voice | "Send the bookcase files to my phone" via assistant |
| USB | Hidden flush-mount USB port on table edge (magnetic cover) |

---

## Data & Backup

| Layer | Detail |
|---|---|
| Primary | Local SSD on the mini PC — fast, works offline |
| Auto-sync | Background sync to home server over Wi-Fi |
| Version control | Git for project files (sketches, 3D models, diagrams) |
| Cloud (optional) | Google Drive / OneDrive as off-site third layer |

---

## Input Devices

| Input | Purpose |
|---|---|
| Huion EMR pen | Drawing, sketching, precise interaction |
| PCAP multi-touch | Gestures, pinch-zoom, multi-finger navigation |
| Virtual assistant | Voice commands, offloads processing to home server (spec in separate session) |
| No physical keyboard/mouse | Touch + pen + voice covers all interaction |

---

## Future Integration

### Virtual Assistant

The table will integrate with the Virtual Assistant project (designed separately). The assistant runs on the table's mini PC and offloads heavy processing to the home server.

| Feature | Detail |
|---|---|
| Voice input | Microphone array (USB, flush-mounted in table) |
| Voice output | Surface transducers |
| Processing | Local mini PC for lightweight tasks, home server for heavy LLM inference |
| Network | Wi-Fi to home server |
| Capabilities | Voice commands for all table functions, 3D model iteration, theme switching, export, map navigation |

---

## Bill of Materials — Estimated Costs

| Component | Estimated Cost |
|---|---|
| Huion Kamvas Pro 27 (144Hz) | ~£1,800 |
| Minisforum 795S7 (RTX 4060, 32GB, 1TB) | ~£700 |
| Motorised standing desk frame | ~£300-500 |
| Linear actuators x5 (4 slides + 1 lift) | ~£200-400 |
| Scissor lift mechanism | ~£100-200 |
| Controller board + wiring + connectors | ~£100-200 |
| Sensors (IR x8+, capacitive touch, reed switch, fingerprint) | ~£100-200 |
| Walnut veneer on birch ply panels (commissioned) | ~£200-400 |
| Matte lacquer finish (commissioned) | ~£100-200 |
| Ceramic coating | ~£30-50 |
| Skirt panels + magnetic mounts | ~£50-100 |
| Rails, magnetic alignment pins, rubber grommets | ~£50-100 |
| Surface transducers + amp board | ~£50-70 |
| LED strips (WS2812B) + controller | ~£30-50 |
| DC power supply + step-down converters | ~£30-50 |
| Power distribution (multi-socket, fuses, wiring) | ~£30-50 |
| Screen protector films (pack) | ~£15-30 |
| Silicone gasket material | ~£10-20 |
| USB extension + flush mount port | ~£10-20 |
| Miscellaneous (cables, cable management, fasteners) | ~£50-100 |
| **Total estimated hardware cost** | **~£3,955 - £5,540** |

Software development is Toby's time — no external cost.

---

## Build Approach

| Aspect | Detail |
|---|---|
| Commissioned work | Top surface panels (cut, veneered, lacquered) — precision joinery and professional finish |
| Self-built | Frame assembly, mechanism, skirt, electronics, software, everything else |
| Phasing | Built in phases but revealed only when fully complete |
| Timeline | No deadline — work on it when time allows |

### Suggested Phases

| Phase | Scope |
|---|---|
| 1 | Physical build — table, motorised frame, sliding mechanism, monitor lift, activation sequence, safety sensors |
| 2 | Sci-fi UI shell — Electron app, theme engine, hub/launcher, drawing canvas, diagram tools |
| 3 | Sketch-to-3D pipeline — parametric engine, 3D viewer, PBR materials, lighting, export |
| 4 | Virtual assistant integration |
