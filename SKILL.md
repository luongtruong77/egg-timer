---
name: egg-timer
description: >
  Build, enhance, or maintain the Egg Hatch Timer web app — a single-file HTML/CSS/JS countdown
  timer where a random baby animal hatches from an egg when the timer expires. Use this skill
  whenever the user mentions the egg timer, egg hatch app, or wants to add features like new
  animals, sounds, backgrounds, UI controls, or themes to this project. Also use when the user
  wants to export, package, or deploy the egg timer.
---

# Egg Hatch Timer — Project Skill

A single-file HTML app (`egg-timer.html`) that combines a countdown timer with an animated egg-hatching reveal of a randomly selected baby animal. Built with vanilla HTML/CSS/JS, no external dependencies beyond Google Fonts.

---

## RESUME — Current State (as of last session)

### Output file
`/mnt/user-data/outputs/egg-timer.html` — the canonical deliverable. Always overwrite this file in place (delete then create, since the tool errors on existing files).

### Version history
| Version | Key changes |
|---------|-------------|
| v1 | Initial build: 20 animals, horizontal slider, basic oscillator tick sound, night sky bg |
| v2 ✅ | 30 animals, 4 themed groups, 4 animated CSS backgrounds, tick-tock sound (BPF noise), vertical dial timer, theme badge |

### What was last built (v2 feature set)
- **30 baby/young animals** split across 4 themes (8 per theme, see `THEMES` object in JS)
- **4 CSS-only animated backgrounds** that fade in on hatch reveal:
  - `#bg-farm` — warm sunrise + grass + wheat/sunflower emoji row
  - `#bg-forest` — deep emerald night + tree emoji row
  - `#bg-wild` — fiery savanna sunset + cactus/palm emoji row
  - `#bg-sky` — blue sky gradient + drifting cloud emoji (`@keyframes drift`)
- **Realistic tick-tock sound** via Web Audio API:
  - Alternating tick (higher ~1100Hz + 2800Hz BPF noise) / tock (lower ~780Hz + 2100Hz BPF noise)
  - Shaped noise burst through bandpass filter → mechanical click character
  - `playTick()` tracks `tickPhase` (even=tick, odd=tock)
- **Vertical dial timer**: ▲/▼ buttons for minutes (0–30) and seconds (0–50, step 10), plus 6 preset pills
- **3-phase egg lifecycle**:
  1. Slow wiggle (`wiggle-slow`, 2s loop)
  2. At 50% → mid cracks appear + fast wiggle (`wiggle-fast`, 0.38s loop) + crack sound
  3. On expire → full crack SVG → `hatch-explode` → animal reveal → themed background transition
- **Theme badge** top of card: shows "🥚 Hatching…" during run, then theme name + tinted bg on reveal
- **Phase dots** (3×) in footer
- **Confetti** (44 pieces) + sparkle burst (8 emojis) on hatch
- **Sound toggle** (tick-tock + crack + hatch jingle)

---

## Architecture

### File structure
Single self-contained HTML file. No build step, no external JS.

```
egg-timer.html
├── <style>           Google Fonts import + all CSS
│   ├── Background layers  (#bg-idle, #bg-farm, #bg-forest, #bg-wild, #bg-sky)
│   ├── Egg SVG animations (wiggle-slow, wiggle-fast, hatch-explode, animal-appear, float-anim)
│   ├── Dial controls       (.dial-box, .dial-btn, .dial-val)
│   └── Confetti / sparkles
├── <body>
│   ├── .bg-layer divs (5 total — idle + 4 themes)
│   ├── .app > h1 + .theme-badge
│   ├── .egg-stage > .egg-wrap > SVG#eggSvg (with #crackMid, #crackFull groups)
│   ├── .animal-reveal#animalReveal
│   ├── .animal-name#animalName
│   └── .controls
│       ├── #timerDisplay + .progress-bar
│       ├── .setter-row (.preset-grid + .setter-divider + .dial-cluster)
│       ├── .btn-row (#startBtn + reset)
│       └── .footer-row (sound toggle + phase dots)
└── <script>
    ├── THEMES object (4 keys: farm/forest/wild/sky, each with name/bg/badge/animals[8])
    ├── State vars
    ├── Audio (getCtx, playTick, playCrack, playHatch)
    ├── Dial logic (syncDial, nudge, setPreset)
    ├── Timer (fmt, updateDisplay)
    ├── Background (setBg)
    ├── pickAnimal
    ├── startTimer / hatchEgg / resetTimer
    └── Effects (spawnSparkles, spawnConfetti, setDot, toggleSound)
```

### Key DOM IDs
| ID | Purpose |
|----|---------|
| `eggSvg` | The egg — className drives animation state |
| `crackMid` | SVG group, opacity 0→1 at 50% |
| `crackFull` | SVG group, opacity 0→1 on hatch |
| `animalReveal` | Emoji div, absolute centered in egg-stage |
| `animalName` | Text below egg |
| `themeBadge` | Pill above egg showing theme |
| `timerDisplay` | Large countdown |
| `progressFill` | Progress bar fill |
| `dialMin` / `dialSec` | Dial value displays |
| `minUp/minDown/secUp/secDown` | Dial buttons (disabled during run) |
| `startBtn` | Disabled during run |
| `dot1/dot2/dot3` | Phase indicator dots |
| `bg-idle/bg-farm/bg-forest/bg-wild/bg-sky` | Background layers |

### CSS animation classes on `eggSvg`
| Class | Effect |
|-------|--------|
| *(none)* | Static |
| `wiggling` | `wiggle-slow` 2s loop |
| `wiggling-fast` | `wiggle-fast` 0.38s loop |
| `hatching` | `hatch-explode` 0.75s forwards |

### Animal reveal classes
| Class | Effect |
|-------|--------|
| `animal-reveal` | Hidden (scale 0, opacity 0) |
| `animal-reveal shown` | `animal-appear` bounce-in animation |
| `animal-reveal floating` | `float-anim` gentle bob (applied 920ms after shown) |

---

## THEMES data structure

```js
const THEMES = {
  farm:   { name:'🌾 Farm',   bg:'bg-farm',   badge:'#e8832a', animals:[8 objects] },
  forest: { name:'🌲 Forest', bg:'bg-forest', badge:'#4a9c2a', animals:[8 objects] },
  wild:   { name:'🦁 Wild',   bg:'bg-wild',   badge:'#e05a1a', animals:[8 objects] },
  sky:    { name:'🌤️ Sky',   bg:'bg-sky',    badge:'#1a6abf', animals:[8 objects] }
};
// Each animal: { emoji: '🐣', name: 'Baby Chick' }
```

---

## Audio design

### Tick-tock (`playTick`)
- Alternates every call via `tickPhase % 2`
- **Tick**: 2800Hz BPF + 1100Hz sine body resonance, gain 0.55/0.14
- **Tock**: 2100Hz BPF + 780Hz sine body resonance, gain 0.38/0.09
- Noise burst: 22ms buffer, envelope `(1 - i/bLen)^4`, run through BPF

### Crack (`playCrack`)
- 200ms noise burst, `exp(-i / (sr * 0.048))` envelope
- 2400Hz bandpass Q=0.9

### Hatch (`playHatch`)
- Calls `playCrack()` + rising sawtooth whoosh (100→1800Hz, 0.1–0.52s)
- 5-note ascending jingle: [523, 659, 784, 1047, 1319] Hz, 0.1s spacing, sine

---

## Suggested next features (backlog ideas)
- [ ] Custom animal name input
- [ ] "Peek" button to preview which animal is in the egg
- [ ] Multiple eggs at once (split-timer mode)
- [ ] Ambient background music per theme (looping, low volume)
- [ ] Mobile haptic feedback on crack/hatch
- [ ] Egg color variants (golden egg = rare animal?)
- [ ] Share/screenshot the hatched animal
- [ ] LocalStorage to remember last timer duration

---

## How to continue in Claude Code (CLI)

1. The output file lives at `/mnt/user-data/outputs/egg-timer.html`
2. To edit: read the file first (`view /mnt/user-data/outputs/egg-timer.html`), then use `str_replace` for targeted edits or delete + `create_file` for full rewrites
3. The file is ~380 lines. Full rewrites are fine — it's self-contained
4. Always test by opening in browser after changes
5. When adding animals: add entries to the relevant `THEMES[key].animals` array (keep 8 per theme for balance, or expand evenly)
6. When adding a new theme: add a `#bg-<name>` CSS layer, a `<div class="bg-layer" id="bg-<name>">` in the HTML, and a new key to `THEMES`

---

## Design tokens
```
Font:       Fredoka One (headings/numbers), Nunito 700/800 (UI)
Night bg:   #1a0a2e → #0c0518 radial
Card bg:    rgba(0,0,0,0.28) with backdrop-filter:blur(16px)
Accent:     #f9e05a (yellow)
Accent2:    #ff7eb3 (pink — urgent state)
Egg:        radial #fffaf2 → #fff5e4 → #e4ccaa
Crack:      #8b6b45 (full), #c8a87a (mid)
```
