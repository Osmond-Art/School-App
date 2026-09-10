# The Data Desk — full build spec for a single-file school reference app

**Purpose of this document:** everything Claude Code needs to build the thing in one go — scope, architecture, colour specs, type, layout, per-tab content, real data payloads, and a copy-paste master prompt. Written for a Year 11–13 NCEA student in Auckland.

---

## 0. Read this before you build (honest scoping)

You asked for "information about everything." That is the one thing that will kill this project, so here is the straight version:

- **A single HTML file that contains everything is a file that loads slowly, is impossible to debug, and that you stop using in week three.** The failure mode is not "too ambitious," it's "12 half-finished tabs, none good enough to beat a Google search."
- **The honest test for every feature: is this faster than searching for it?** A periodic table you can filter and see trends on — yes, clearly faster. A "biology notes" tab that's just paragraphs you'll never re-read — no, that's a worse version of your own notes.
- **So: build depth in four tabs, not breadth in twelve.** Tabs 1–4 below (Periodic Table, Chemistry, Physics, Maths) are genuinely better as a tool than as a search. The rest are listed as optional Phase 3 — build them only if you're still opening the app daily after two weeks.
- **Size budget: 500 KB, hard.** All 118 elements with full data is ~40 KB of JSON. Everything else is code. If you blow the budget, you've added prose that belongs in your notes app.

Build order is in §7. Don't skip to the fun parts.

---

## 1. What it is

**Name:** The Data Desk (rename freely — but pick something that isn't "StudyHub").

**One line:** An offline single-file reference desk for NCEA science and maths — the lookups you need mid-homework, answerable in under five seconds.

**Primary job:** Lookup speed. Secondary job: interactive tools that do arithmetic you'd otherwise do wrong at 11pm.

**Audience:** You. Not a class, not a product. Design for one person who knows where everything is.

---

## 2. Technical constraints (non-negotiable)

| Constraint | Rule |
|---|---|
| Output | One file: `datadesk.html`. HTML + CSS + JS inline. |
| Dependencies | Zero runtime dependencies. No React, no Tailwind CDN, no chart library. Vanilla JS, ES2020. |
| Fonts | One optional `<link>` to Google Fonts with a full system fallback stack. The file must look correct with no internet. |
| Persistence | `localStorage` only, namespaced `dd:`. Never lose user data on a schema change — version the schema and migrate. |
| Offline | Must work fully offline after first load. No fetch calls at runtime. |
| Print | `@media print` — periodic table and formula sheets print to one clean A4 page each. This is the underrated feature. |
| Browser | Chrome/Edge/Safari current. No IE shims, no build step. |
| Performance | First paint under 400 ms. No layout thrash on tab switch. |
| Accessibility | Full keyboard nav, visible focus rings, `prefers-reduced-motion` respected, 4.5:1 text contrast minimum. |

**Code structure inside the file** (in this order):

```
<head>
  <meta> / <title>
  <style>            ← tokens, base, layout, components, tab-specific, print
</head>
<body>
  <header>           ← command bar + tab rail
  <main id="desk">   ← one <section class="panel" data-tab="..."> per tab, hidden by default
  <template>s        ← element detail card, flashcard, modal
  <script>
    DATA = {...}     ← all static payloads, frozen
    store  = {...}   ← localStorage wrapper w/ schema version
    router = {...}   ← hash routing (#periodic, #chem/molar)
    search = {...}   ← global index, built once on load
    tabs   = {...}   ← one init() per tab, lazy: run on first activation
  </script>
</body>
```

**Lazy init matters.** Don't build the DOM for all tabs at load. Each tab exposes `init()` that runs the first time it's shown, then a flag stops it re-running.

---

## 3. Design system

### 3.1 Direction

The visual world is the **NZQA resource booklet and the school lab bench** — graph paper, printed data tables, flame tests, engineering drawings. Not "edtech." Not a SaaS dashboard.

Two decisions define the look:

1. **The ground is graph paper, not a card.** A faint 5 mm grid drawn in CSS underlies the whole desk. Content sits *on* it in flat panels with hairline rules — not in floating rounded cards with drop shadows.
2. **The accent palette is the flame test series.** Every semantic colour in the app is a real flame emission colour — lithium crimson, sodium amber, copper-green, potassium lilac, strontium red. This is where the subject-matter grounding comes from, and it gives you six colour-coded subject areas for free.

**Deliberately avoided:** cream/terracotta editorial, near-black + acid-green, uniform rounded cards with soft grey shadows, ALL-CAPS eyebrow labels, "→" glued to button text, meta strings joined with middle dots. If a draft starts producing those, it's drifting to default.

### 3.2 Colour tokens

```css
:root{
  /* ground */
  --paper:        #F3F5F4;  /* cool grey-green paper, not cream */
  --paper-sunk:   #E8EDEB;  /* recessed wells, table stripes */
  --grid:         #D6DEDB;  /* 5mm graph rule, ~8% opacity in use */
  --rule:         #C2CCC9;  /* hairline borders */
  --rule-strong:  #8A9A96;

  /* ink */
  --ink:          #16232E;  /* deep slate-blue, primary text */
  --ink-2:        #47585F;  /* secondary text, captions */
  --ink-3:        #7C8C90;  /* disabled, placeholder */

  /* flame-test accents — semantic */
  --signal:       #0F6E63;  /* copper flame. primary action, links, focus */
  --signal-lift:  #12897C;  /* hover */
  --signal-wash:  #DCEBE8;  /* selected row, tint fills */
  --crimson:      #B3123A;  /* lithium. errors, hazards, exothermic, negatives */
  --amber:        #C7860B;  /* sodium. warnings, highlights, "check this" */
  --lilac:        #6E5AC8;  /* potassium. secondary tool accent, maths */
  --strontium:    #D4453B;  /* strontium. reserved: periodic table trend hot end */
  --barium:       #4E9A2F;  /* barium. reserved: trend cool end, correct answers */
}

@media (prefers-color-scheme: dark){
  :root{
    --paper:      #10262A;  /* deep petrol — a chosen hue, not a black stand-in */
    --paper-sunk: #0B1D21;
    --grid:       #1C3A3F;
    --rule:       #24474D;
    --rule-strong:#3E6B71;
    --ink:        #E4EFEC;
    --ink-2:      #A3BAB6;
    --ink-3:      #6E8A87;
    --signal:     #4FC7B6;
    --signal-lift:#6FDCCB;
    --signal-wash:#123B3B;
    --crimson:    #FF7A93;
    --amber:      #F0B44A;
    --lilac:      #A996F0;
    --strontium:  #FF7A6B;
    --barium:     #86D45F;
  }
}
```

**Dark mode is a real requirement** — you will use this at night. Follow system preference by default with a manual override stored in `dd:theme`.

### 3.3 Periodic table category colours

These are the only place a big spread of hues is allowed. Tinted fills, ink-coloured text — never white text on a saturated tile (it kills contrast at small sizes and prints as mud).

```css
--cat-alkali:        #F6D9D0;  /* fill */   --cat-alkali-edge:      #C25A3E;
--cat-alkaline:      #F8E7C8;               --cat-alkaline-edge:    #C08A22;
--cat-transition:    #DCE7EC;               --cat-transition-edge:  #547A8E;
--cat-post-trans:    #E2E6DE;               --cat-post-trans-edge:  #6E7B63;
--cat-metalloid:     #E5DFF2;               --cat-metalloid-edge:   #6E5AC8;
--cat-nonmetal:      #D9EDDF;               --cat-nonmetal-edge:    #35774A;
--cat-halogen:       #D6EAF0;               --cat-halogen-edge:     #2C7C93;
--cat-noble:         #EDDDE8;               --cat-noble-edge:       #8E4A78;
--cat-lanthanide:    #F1E3D5;               --cat-lanthanide-edge:  #A9722F;
--cat-actinide:      #EEDCDC;               --cat-actinide-edge:    #9C4444;
--cat-unknown:       #E4E7E8;               --cat-unknown-edge:     #7C8C90;
```

Dark mode: keep the same edge hues, replace fills with `color-mix(in oklab, <edge> 22%, var(--paper))`.

### 3.4 Typography

Two families. Numbers get tabular figures rather than a third monospace family.

```css
--font-ui:   "Archivo", "Segoe UI Variable", "Segoe UI", system-ui, -apple-system, sans-serif;
--font-read: "Source Serif 4", Georgia, "Times New Roman", serif;
```

- **Archivo** — everything structural: nav, tables, element tiles, buttons, data. It's a grotesque with enough width variation to set condensed table headers without looking like Helvetica.
- **Source Serif 4** — explanation prose only (the "why this formula works" text, definitions). Max 68 characters per line, `line-height: 1.6`.
- All numeric data: `font-variant-numeric: tabular-nums lining-nums;` — non-negotiable in tables and the periodic table.

Scale (1.25 minor third, 16 px base):

```
--t-xs:  0.75rem   /* 12px — element mass, table footnotes */
--t-sm:  0.875rem  /* 14px — table body, tool labels */
--t-base:1rem      /* 16px — body */
--t-md:  1.25rem   /* 20px — panel headings */
--t-lg:  1.563rem  /* 25px — tab titles */
--t-xl:  1.953rem  /* 31px — element symbol in detail card */
--t-2xl: 3.052rem  /* 49px — the one display moment: selected element symbol */
```

Weights: 400 body, 500 UI labels, 600 headings, 700 reserved for the element symbol. Don't use more than four.

### 3.5 Space, rules, radius

```css
--s1:4px; --s2:8px; --s3:12px; --s4:16px; --s5:24px; --s6:32px; --s7:48px;
--radius-tile: 2px;   /* element tiles — near-square, like printed cells */
--radius-panel: 0;    /* panels are flat sheets on the grid */
--radius-control: 6px;/* inputs and buttons only */
```

Hierarchy is carried by **rule weight**, not shadow:
- `1px solid var(--rule)` — internal divisions
- `2px solid var(--ink)` — the active panel's top edge and the command bar underline
- Shadows: exactly one, on the floating element-detail card: `0 8px 28px rgba(22,35,46,.16)`. Nowhere else.

### 3.6 Layout

```
┌──────────────────────────────────────────────────────────────────┐
│  DATA DESK        [ search anything — press / ]        ☾  ⌘K     │  56px, sticky
├────┬─────────────────────────────────────────────────────────────┤
│ Pe │                                                             │
│ Ch │   panel title                          [tool controls]      │
│ Ph │   ───────────────────────────────────────────────────       │
│ Ma │                                                             │
│ Bi │   ← content region (graph-paper ground shows through)       │
│ En │                                                             │
│ St │                                                             │
│    │                                                             │
│ ⚙  │                                                             │
└────┴─────────────────────────────────────────────────────────────┘
  64px rail                          fluid, max-width 1400px
```

- **Left rail**, 64 px, icon + 2-letter label stacked, vertical. Active tab: 3 px left bar in `--signal` and a `--paper` background that visually connects it to the panel. Each subject carries its own accent (see §4) — the rail is the only place all six accents appear together.
- **Command bar is the hero.** Not a logo, not a hero heading. On load, focus sits in the search field, and the empty state below shows the periodic table at reduced scale as a live "map." That's the most characteristic object in the subject's world, and it doubles as the default tab.
- **Mobile (<720px):** rail becomes a bottom bar, 5 items + overflow. Periodic table switches to a vertically scrolling list grouped by category, with a "wide table" toggle that allows horizontal scroll. Do not try to squeeze 18 columns onto a phone.

### 3.7 Motion

One orchestrated moment: on first load, the periodic table tiles fade in over 220 ms in a single stagger by atomic number (2 ms per element, capped at 240 ms total). That's it.

Everything else is response-only: 120 ms `ease-out` on hover tint, 160 ms on the detail card opening, instant on tab switch. All of it inside `@media (prefers-reduced-motion: no-preference)`.

### 3.8 Copy rules

- Sentence case everywhere. No ALL-CAPS labels.
- Buttons name the outcome: "Balance equation", "Save note", "Reset timer".
- Empty states give a next action: "No flashcards yet. Add one, or import a deck." Not "Nothing here!"
- Errors say what and how: "Equation can't balance — check the formula on the left side." Never "Invalid input."
- No exclamation marks, no emoji in UI chrome.

---

## 4. Tab-by-tab specification

Each tab owns one accent, used for its rail marker, active headings, and tool highlights.

### Tab 1 — Periodic Table  ·  accent `--signal`  ·  route `#periodic`

The centrepiece. Everything else is judged against how good this is.

**Layout:** true 18×7 grid plus the lanthanide/actinide rows offset below with a connecting bracket. `display:grid; grid-template-columns:repeat(18,1fr); gap:3px;` Tiles are `aspect-ratio:1` with a min of 44 px so they stay tappable.

**Tile contents:** atomic number (top-left, `--t-xs`, `--ink-2`), symbol (centre, 600 weight, `--t-md`), name (below, `--t-xs`, truncated with ellipsis), mass (bottom, `--t-xs`, tabular). Below 56 px tile width, drop the name; below 44 px, drop the mass.

**Controls row above the table:**
- Search field — filters by name, symbol, or atomic number as you type. Non-matches drop to 15% opacity, matches keep full colour. Never remove tiles from the grid; the shape of the table is the information.
- Colour-by selector: `Category · State at 25 °C · Group block (s/p/d/f) · Metal–nonmetal`
- Trend overlay selector: `None · Atomic radius · Electronegativity · First ionisation energy · Melting point · Electron affinity · Discovery year`. Trend mode replaces category fills with a two-stop scale from `--barium` (low) to `--strontium` (high), shows a legend with real units, and greys elements with no data. **This is the single feature that makes the app better than a printed table** — build it properly.
- Temperature slider, 0–6000 K, that recolours tiles by predicted state (solid / liquid / gas) using the melting and boiling points. Cheap to build, genuinely impressive, and useful for state-change questions.

**Element detail card:** opens on click, floats over the table (the one shadow in the app), closes on Esc or outside-click, and is deep-linkable at `#periodic/Fe`.

Card contents, in order: symbol at `--t-2xl` on a category-tinted ground · name, atomic number, standard atomic weight · category, group, period, block · electron configuration in full and in noble-gas shorthand · shell counts (2,8,14,2) with a small shell diagram drawn in SVG · common oxidation states · electronegativity (Pauling) · atomic radius (pm) · first ionisation energy (kJ/mol) · melting and boiling points in °C and K · density · phase at STP · discovery year and discoverer · one sentence on where it's actually used. Then a "Copy data" button that puts a tab-separated row on the clipboard for pasting into a lab report.

**Keyboard:** arrow keys move between tiles by grid position, Enter opens the card, Esc closes, `/` jumps back to search.

**Print:** the table alone, one A4 landscape page, category fills preserved, controls hidden.

---

### Tab 2 — Chemistry  ·  accent `--crimson`  ·  route `#chem`

Sub-sections as a segmented control across the top, each deep-linkable (`#chem/molar`).

1. **Molar calculator** — enter a formula (`Ca(OH)2`, `CuSO4.5H2O`), get molar mass with a per-element breakdown table showing count × atomic mass = subtotal. Then a mass ⇄ moles ⇄ particles converter that fills in whichever field you leave blank. Must handle nested brackets and hydrate dots.
2. **Equation balancer** — enter `Fe + O2 -> Fe2O3`, get balanced coefficients. Implement as Gaussian elimination over rationals on the element-count matrix; don't brute-force coefficients. Show the balanced equation with state symbols left blank for the user to add, and flag when no integer solution exists.
3. **Solubility and precipitate table** — the NCEA solubility rules as an interactive cation × anion grid. Click a cell: soluble / insoluble, plus the precipitate colour where relevant. Colour is what gets asked in exams.
4. **Reference sheets** — polyatomic ions (§5.3), reactivity series, common acids and bases with formulas, strong vs weak, the activity series with displacement notes, flame test colours (with actual swatches drawn from your accent palette — that's why the palette exists).
5. **Organic chemistry** — functional group table: name, general formula, suffix/prefix, an SVG skeletal sketch, one example. Plus a naming quick-reference for alkanes/alkenes/alkanols/alkanoic acids up to 10 carbons.
6. **Titration helper** — c₁V₁/n₁ = c₂V₂/n₂ solver with mole-ratio input, showing the working line by line, not just the answer. Seeing the substitution step is the point.

---

### Tab 3 — Physics  ·  accent `--amber`  ·  route `#phys`

1. **Formula sheet**, grouped: mechanics, waves, electricity, atomic/nuclear, energy. Each entry: formula in proper notation, every symbol defined with units, and one line on when it applies (and when it doesn't — "constant acceleration only" saves more marks than the formula does).
2. **Formula solver** — pick a formula, fill any n−1 variables, get the missing one with units carried through and the rearrangement shown. Cover at minimum: v = u + at, v² = u² + 2as, s = ut + ½at², F = ma, p = mv, W = Fd, Ek = ½mv², Ep = mgh, P = W/t, V = IR, P = VI, Q = It, f = 1/T, v = fλ, n₁sinθ₁ = n₂sinθ₂, E = hf.
3. **Constants table** — §5.2, with a copy button on each value.
4. **Unit converter** — SI prefixes yotta→yocto, plus the conversions you actually need: km/h ⇄ m/s, °C ⇄ K ⇄ °F, J ⇄ eV ⇄ kWh, N ⇄ kg·f, Pa ⇄ atm ⇄ bar ⇄ mmHg.
5. **Circuit calculator** — series/parallel resistance for an arbitrary list, plus voltage divider and total current. Text input, not a drag-and-drop circuit builder; the builder is a month of work for less value.
6. **Significant figures and uncertainty** — enter a value with uncertainty, get correct sig figs and propagated error for +, −, ×, ÷, and powers. Almost nobody builds this and it's where marks quietly disappear.

---

### Tab 4 — Maths  ·  accent `--lilac`  ·  route `#maths`

1. **Formula reference** — algebra identities, quadratic formula and discriminant cases, log and index laws, sequences and series, binomial expansion, geometry (area/volume/surface area with a small SVG per shape), coordinate geometry, trig identities, differentiation and integration rules with a short table of standards.
2. **Unit circle** — interactive. Drag around the circle, see the angle in degrees and radians, exact sin/cos/tan values as surds, and the coordinate. This beats memorising it.
3. **Graphing tool** — plot up to three functions on a canvas, with pan and zoom, an x/y window, and a readout of intercepts and turning points where they're computable. Parse expressions with a small shunting-yard parser you write yourself (~120 lines). **Never use `eval()`** — write the tokeniser.
4. **Solvers** — quadratic (with exact surd form, not just decimals), simultaneous equations 2×2 and 3×3, and a triangle solver (sine rule, cosine rule, area, with ambiguous-case warning).
5. **Statistics** — paste a data column, get mean, median, mode, range, quartiles, IQR, standard deviation (sample and population, correctly distinguished), plus a box plot and a histogram drawn on canvas.

---

### Phase 3 — optional tabs (only after two weeks of daily use)

- **Biology** `--barium` — codon table (§5.6), Punnett square generator for mono/dihybrid crosses, labelled SVG diagrams (animal cell, plant cell, heart, nephron, neuron), enzyme and transport summary tables.
- **English & writing** — paragraph structures (TEEL/SEXY), literary and rhetorical device glossary with examples, essay planning scaffold, and an APA 7 citation builder (book, website, journal, film) that outputs a formatted reference to the clipboard.
- **Study** — Pomodoro timer that survives tab switches, spaced-repetition flashcards (SM-2 lite, stored in localStorage, with JSON export), an NCEA credit tracker (standard code, level, credits, achieved/merit/excellence, running totals against 80 credits and endorsement thresholds), and an exam countdown.
- **Scratchpad** — plain-text notes with autosave, plus a scientific calculator that keeps a visible history tape.

---

### Global features (build these in Phase 1, they're the spine)

- **Command palette**, `Ctrl/Cmd-K` or `/`. Searches across everything: element names and symbols, formula names, constants, polyatomic ions, glossary terms, and tab names. Results grouped by source with the tab's accent as a left border. Enter jumps straight to the item, deep-linked. Build the index once at load from the `DATA` object — one flat array of `{label, keywords, tab, route}` — and match with simple substring + prefix scoring. No fuzzy library.
- **Hash routing** — every panel and sub-panel has a URL. `#periodic/Fe`, `#chem/molar`, `#maths/unit-circle`. This is what makes it bookmarkable, which is what makes you use it.
- **Theme toggle** — system / light / dark, persisted.
- **Keyboard map** — `?` opens a shortcut sheet. `1`–`6` switch tabs. Esc closes anything open.

---

## 5. Data payloads

Paste these straight into the `DATA` object. Everything below is real data, not placeholder.

### 5.1 Periodic table — all 118 elements

Compact array-of-arrays to keep the file small. Schema:

```js
// [Z, symbol, name, standardAtomicWeight, category, group, period]
// mass in square-bracket string = mass number of the most stable known isotope
```

Categories map to the `--cat-*` tokens in §3.3: `alkali, alkaline, transition, post-transition, metalloid, nonmetal, halogen, noble, lanthanide, actinide`.

```js
const ELEMENTS = [
[1,"H","Hydrogen",1.008,"nonmetal",1,1],
[2,"He","Helium",4.003,"noble",18,1],
[3,"Li","Lithium",6.94,"alkali",1,2],
[4,"Be","Beryllium",9.012,"alkaline",2,2],
[5,"B","Boron",10.81,"metalloid",13,2],
[6,"C","Carbon",12.011,"nonmetal",14,2],
[7,"N","Nitrogen",14.007,"nonmetal",15,2],
[8,"O","Oxygen",15.999,"nonmetal",16,2],
[9,"F","Fluorine",18.998,"halogen",17,2],
[10,"Ne","Neon",20.180,"noble",18,2],
[11,"Na","Sodium",22.990,"alkali",1,3],
[12,"Mg","Magnesium",24.305,"alkaline",2,3],
[13,"Al","Aluminium",26.982,"post-transition",13,3],
[14,"Si","Silicon",28.085,"metalloid",14,3],
[15,"P","Phosphorus",30.974,"nonmetal",15,3],
[16,"S","Sulfur",32.06,"nonmetal",16,3],
[17,"Cl","Chlorine",35.45,"halogen",17,3],
[18,"Ar","Argon",39.95,"noble",18,3],
[19,"K","Potassium",39.098,"alkali",1,4],
[20,"Ca","Calcium",40.078,"alkaline",2,4],
[21,"Sc","Scandium",44.956,"transition",3,4],
[22,"Ti","Titanium",47.867,"transition",4,4],
[23,"V","Vanadium",50.942,"transition",5,4],
[24,"Cr","Chromium",51.996,"transition",6,4],
[25,"Mn","Manganese",54.938,"transition",7,4],
[26,"Fe","Iron",55.845,"transition",8,4],
[27,"Co","Cobalt",58.933,"transition",9,4],
[28,"Ni","Nickel",58.693,"transition",10,4],
[29,"Cu","Copper",63.546,"transition",11,4],
[30,"Zn","Zinc",65.38,"transition",12,4],
[31,"Ga","Gallium",69.723,"post-transition",13,4],
[32,"Ge","Germanium",72.630,"metalloid",14,4],
[33,"As","Arsenic",74.922,"metalloid",15,4],
[34,"Se","Selenium",78.971,"nonmetal",16,4],
[35,"Br","Bromine",79.904,"halogen",17,4],
[36,"Kr","Krypton",83.798,"noble",18,4],
[37,"Rb","Rubidium",85.468,"alkali",1,5],
[38,"Sr","Strontium",87.62,"alkaline",2,5],
[39,"Y","Yttrium",88.906,"transition",3,5],
[40,"Zr","Zirconium",91.224,"transition",4,5],
[41,"Nb","Niobium",92.906,"transition",5,5],
[42,"Mo","Molybdenum",95.95,"transition",6,5],
[43,"Tc","Technetium","[98]","transition",7,5],
[44,"Ru","Ruthenium",101.07,"transition",8,5],
[45,"Rh","Rhodium",102.91,"transition",9,5],
[46,"Pd","Palladium",106.42,"transition",10,5],
[47,"Ag","Silver",107.87,"transition",11,5],
[48,"Cd","Cadmium",112.41,"transition",12,5],
[49,"In","Indium",114.82,"post-transition",13,5],
[50,"Sn","Tin",118.71,"post-transition",14,5],
[51,"Sb","Antimony",121.76,"metalloid",15,5],
[52,"Te","Tellurium",127.60,"metalloid",16,5],
[53,"I","Iodine",126.90,"halogen",17,5],
[54,"Xe","Xenon",131.29,"noble",18,5],
[55,"Cs","Caesium",132.91,"alkali",1,6],
[56,"Ba","Barium",137.33,"alkaline",2,6],
[57,"La","Lanthanum",138.91,"lanthanide",3,6],
[58,"Ce","Cerium",140.12,"lanthanide",null,6],
[59,"Pr","Praseodymium",140.91,"lanthanide",null,6],
[60,"Nd","Neodymium",144.24,"lanthanide",null,6],
[61,"Pm","Promethium","[145]","lanthanide",null,6],
[62,"Sm","Samarium",150.36,"lanthanide",null,6],
[63,"Eu","Europium",151.96,"lanthanide",null,6],
[64,"Gd","Gadolinium",157.25,"lanthanide",null,6],
[65,"Tb","Terbium",158.93,"lanthanide",null,6],
[66,"Dy","Dysprosium",162.50,"lanthanide",null,6],
[67,"Ho","Holmium",164.93,"lanthanide",null,6],
[68,"Er","Erbium",167.26,"lanthanide",null,6],
[69,"Tm","Thulium",168.93,"lanthanide",null,6],
[70,"Yb","Ytterbium",173.05,"lanthanide",null,6],
[71,"Lu","Lutetium",174.97,"lanthanide",null,6],
[72,"Hf","Hafnium",178.49,"transition",4,6],
[73,"Ta","Tantalum",180.95,"transition",5,6],
[74,"W","Tungsten",183.84,"transition",6,6],
[75,"Re","Rhenium",186.21,"transition",7,6],
[76,"Os","Osmium",190.23,"transition",8,6],
[77,"Ir","Iridium",192.22,"transition",9,6],
[78,"Pt","Platinum",195.08,"transition",10,6],
[79,"Au","Gold",196.97,"transition",11,6],
[80,"Hg","Mercury",200.59,"transition",12,6],
[81,"Tl","Thallium",204.38,"post-transition",13,6],
[82,"Pb","Lead",207.2,"post-transition",14,6],
[83,"Bi","Bismuth",208.98,"post-transition",15,6],
[84,"Po","Polonium","[209]","post-transition",16,6],
[85,"At","Astatine","[210]","halogen",17,6],
[86,"Rn","Radon","[222]","noble",18,6],
[87,"Fr","Francium","[223]","alkali",1,7],
[88,"Ra","Radium","[226]","alkaline",2,7],
[89,"Ac","Actinium","[227]","actinide",3,7],
[90,"Th","Thorium",232.04,"actinide",null,7],
[91,"Pa","Protactinium",231.04,"actinide",null,7],
[92,"U","Uranium",238.03,"actinide",null,7],
[93,"Np","Neptunium","[237]","actinide",null,7],
[94,"Pu","Plutonium","[244]","actinide",null,7],
[95,"Am","Americium","[243]","actinide",null,7],
[96,"Cm","Curium","[247]","actinide",null,7],
[97,"Bk","Berkelium","[247]","actinide",null,7],
[98,"Cf","Californium","[251]","actinide",null,7],
[99,"Es","Einsteinium","[252]","actinide",null,7],
[100,"Fm","Fermium","[257]","actinide",null,7],
[101,"Md","Mendelevium","[258]","actinide",null,7],
[102,"No","Nobelium","[259]","actinide",null,7],
[103,"Lr","Lawrencium","[266]","actinide",null,7],
[104,"Rf","Rutherfordium","[267]","transition",4,7],
[105,"Db","Dubnium","[268]","transition",5,7],
[106,"Sg","Seaborgium","[269]","transition",6,7],
[107,"Bh","Bohrium","[270]","transition",7,7],
[108,"Hs","Hassium","[269]","transition",8,7],
[109,"Mt","Meitnerium","[278]","transition",9,7],
[110,"Ds","Darmstadtium","[281]","transition",10,7],
[111,"Rg","Roentgenium","[282]","transition",11,7],
[112,"Cn","Copernicium","[285]","transition",12,7],
[113,"Nh","Nihonium","[286]","post-transition",13,7],
[114,"Fl","Flerovium","[289]","post-transition",14,7],
[115,"Mc","Moscovium","[290]","post-transition",15,7],
[116,"Lv","Livermorium","[293]","post-transition",16,7],
[117,"Ts","Tennessine","[294]","halogen",17,7],
[118,"Og","Oganesson","[294]","noble",18,7]
];
```

**Extended fields — Claude Code must add these**, keyed by symbol, because the trend overlays and the detail card depend on them:

```js
// EXT["Fe"] = {
//   config: "1s² 2s² 2p⁶ 3s² 3p⁶ 3d⁶ 4s²",
//   shorthand: "[Ar] 3d⁶ 4s²",
//   shells: [2,8,14,2],
//   ox: [+2,+3],                 // common oxidation states
//   en: 1.83,                    // Pauling electronegativity, null if none
//   radius: 126,                 // empirical atomic radius, pm
//   ie1: 762.5,                  // first ionisation energy, kJ/mol
//   mp: 1811, bp: 3134,          // K
//   density: 7.874,              // g/cm³ at 20 °C
//   phase: "solid",              // at 25 °C, 1 atm
//   discovered: -3000, by: "Ancient civilisations",
//   use: "Structural steel, haemoglobin."
// }
```

**Accuracy instruction for Claude Code:** generate `EXT` for all 118 from your own knowledge, but mark any field you're not confident about as `null` rather than guessing. A missing value renders as "—" and greys out in trend mode; a wrong value teaches Osmond the wrong thing. Masses above are IUPAC-consistent to 5 significant figures — worth spot-checking a dozen against a current IUPAC table before shipping.

### 5.2 Physical constants

| Quantity | Symbol | Value | Unit |
|---|---|---|---|
| Speed of light in vacuum | c | 2.998 × 10⁸ | m s⁻¹ |
| Planck constant | h | 6.626 × 10⁻³⁴ | J s |
| Planck constant | h | 4.136 × 10⁻¹⁵ | eV s |
| Elementary charge | e | 1.602 × 10⁻¹⁹ | C |
| Electron mass | mₑ | 9.109 × 10⁻³¹ | kg |
| Proton mass | m_p | 1.673 × 10⁻²⁷ | kg |
| Neutron mass | m_n | 1.675 × 10⁻²⁷ | kg |
| Unified atomic mass unit | u | 1.661 × 10⁻²⁷ | kg |
| Avogadro constant | N_A | 6.022 × 10²³ | mol⁻¹ |
| Molar gas constant | R | 8.314 | J K⁻¹ mol⁻¹ |
| Boltzmann constant | k_B | 1.381 × 10⁻²³ | J K⁻¹ |
| Gravitational constant | G | 6.674 × 10⁻¹¹ | N m² kg⁻² |
| Free-fall acceleration (NZ, ~41°S) | g | 9.81 | m s⁻² |
| Molar volume, ideal gas at STP (0 °C, 100 kPa) | V_m | 22.7 | L mol⁻¹ |
| Molar volume, ideal gas at 25 °C, 100 kPa | V_m | 24.8 | L mol⁻¹ |
| Permittivity of free space | ε₀ | 8.854 × 10⁻¹² | F m⁻¹ |
| Coulomb constant | k | 8.988 × 10⁹ | N m² C⁻² |
| Permeability of free space | μ₀ | 1.257 × 10⁻⁶ | T m A⁻¹ |
| Stefan–Boltzmann constant | σ | 5.670 × 10⁻⁸ | W m⁻² K⁻⁴ |
| Faraday constant | F | 9.649 × 10⁴ | C mol⁻¹ |
| Standard atmosphere | atm | 1.013 × 10⁵ | Pa |
| Ionic product of water at 25 °C | K_w | 1.00 × 10⁻¹⁴ | mol² L⁻² |
| Specific heat capacity of water | c | 4.18 × 10³ | J kg⁻¹ K⁻¹ |
| Latent heat of fusion, water | L_f | 3.34 × 10⁵ | J kg⁻¹ |
| Latent heat of vaporisation, water | L_v | 2.26 × 10⁶ | J kg⁻¹ |
| Earth mass | M_E | 5.972 × 10²⁴ | kg |
| Earth radius (mean) | R_E | 6.371 × 10⁶ | m |

### 5.3 Polyatomic ions

**1−:** hydroxide OH⁻ · nitrate NO₃⁻ · nitrite NO₂⁻ · hydrogencarbonate HCO₃⁻ · hydrogensulfate HSO₄⁻ · dihydrogenphosphate H₂PO₄⁻ · ethanoate (acetate) CH₃COO⁻ · permanganate MnO₄⁻ · cyanide CN⁻ · hypochlorite ClO⁻ · chlorate ClO₃⁻ · perchlorate ClO₄⁻ · thiocyanate SCN⁻ · hydride H⁻

**2−:** carbonate CO₃²⁻ · sulfate SO₄²⁻ · sulfite SO₃²⁻ · chromate CrO₄²⁻ · dichromate Cr₂O₇²⁻ · hydrogenphosphate HPO₄²⁻ · thiosulfate S₂O₃²⁻ · oxalate C₂O₄²⁻ · peroxide O₂²⁻

**3−:** phosphate PO₄³⁻ · phosphite PO₃³⁻ · nitride N³⁻

**1+:** ammonium NH₄⁺ · hydronium H₃O⁺

### 5.4 SI prefixes

Y 10²⁴ · Z 10²¹ · E 10¹⁸ · P 10¹⁵ · T 10¹² · G 10⁹ · M 10⁶ · k 10³ · h 10² · da 10¹ · d 10⁻¹ · c 10⁻² · m 10⁻³ · µ 10⁻⁶ · n 10⁻⁹ · p 10⁻¹² · f 10⁻¹⁵ · a 10⁻¹⁸ · z 10⁻²¹ · y 10⁻²⁴

### 5.5 Solubility rules (NCEA form)

Soluble: all nitrates; all Group 1 and ammonium salts; all ethanoates; most chlorides, bromides, iodides **except** Ag⁺, Pb²⁺, Hg₂²⁺; most sulfates **except** Ba²⁺, Pb²⁺, Ca²⁺ (slightly), Sr²⁺.

Insoluble: most carbonates, phosphates, sulfides, hydroxides, **except** Group 1 and ammonium. Ca(OH)₂ and Ba(OH)₂ are slightly soluble.

Precipitate colours worth memorising: AgCl white · AgBr cream · AgI yellow · PbI₂ bright yellow · BaSO₄ white · Cu(OH)₂ pale blue · Fe(OH)₂ green · Fe(OH)₃ rust brown · CuCO₃ green-blue · CaCO₃ white.

Flame tests: Li crimson · Na intense yellow-orange · K lilac · Ca brick red · Sr scarlet · Ba apple green · Cu blue-green.

### 5.6 Codon table (mRNA → amino acid)

Store as a 64-key object. Start: AUG (Met). Stops: UAA, UAG, UGA.

Phe UUU UUC · Leu UUA UUG CUU CUC CUA CUG · Ile AUU AUC AUA · Met AUG · Val GUU GUC GUA GUG · Ser UCU UCC UCA UCG AGU AGC · Pro CCU CCC CCA CCG · Thr ACU ACC ACA ACG · Ala GCU GCC GCA GCG · Tyr UAU UAC · His CAU CAC · Gln CAA CAG · Asn AAU AAC · Lys AAA AAG · Asp GAU GAC · Glu GAA GAG · Cys UGU UGC · Trp UGG · Arg CGU CGC CGA CGG AGA AGG · Gly GGU GGC GGA GGG · Stop UAA UAG UGA

---

## 6. Storage schema

```js
// localStorage, all keys prefixed dd:
"dd:v"        → "1"                      // schema version — migrate, never wipe
"dd:theme"    → "system" | "light" | "dark"
"dd:lastTab"  → "#periodic/Fe"
"dd:recent"   → [{label, route, at}]     // last 12 lookups, shown in the empty command palette
"dd:pinned"   → ["#chem/molar", ...]     // pinned tools, shown first in the palette
"dd:notes"    → "..."                    // scratchpad text
"dd:cards"    → [{id, front, back, ease, due, deck}]
"dd:credits"  → [{code, level, credits, grade, subject}]
```

On load, if `dd:v` is missing, seed defaults. If it's lower than current, run the migration chain — never `localStorage.clear()`.

---

## 7. Build order

Do not build tabs in parallel. Each phase must run and be usable before the next starts.

**Phase 1 — the shell.** Tokens, graph-paper ground, rail, command bar, hash router, theme toggle, keyboard map, print stylesheet, storage wrapper. One dummy tab. *Test: routes work, theme persists, refresh restores position.*

**Phase 2 — the periodic table, completely.** All 118 tiles, the `EXT` dataset, search filter, colour-by, trend overlays, temperature slider, detail card, keyboard grid nav, print layout. *Test: every element opens, no `undefined` anywhere, prints cleanly.*

**Phase 3 — Chemistry.** Molar calculator and equation balancer first — those are real algorithms and need proper testing against known cases. Then the reference sheets.

**Phase 4 — Physics.** Formula sheet, solver, constants, converter, sig figs.

**Phase 5 — Maths.** Reference, unit circle, expression parser, graphing, solvers, stats.

**Phase 6 — global search index across everything built so far.** Then stop and use it for two weeks before considering Phase 3 optional tabs.

---

## 8. Master prompt — paste this into Claude Code

> Build `datadesk.html`, a single self-contained HTML file: an offline reference desk for an NCEA Level 1–3 student in New Zealand. Attached is the full spec — follow it exactly, and treat §2 (constraints), §3 (design system), and §5 (data) as fixed.
>
> Work in phases per §7. **Stop after each phase and tell me what to test.** Do not start the next phase until I confirm.
>
> Hard rules:
> - One file. Vanilla JS, no frameworks, no CDN dependencies except one optional Google Fonts `<link>` with full system fallback. The file must work offline.
> - Use the exact colour tokens and type scale in §3. Don't substitute your own palette. Specifically avoid: cream backgrounds with terracotta accents, near-black grounds with acid-green accents, uniform rounded cards with soft grey shadows, ALL-CAPS eyebrow labels, arrows appended to button text.
> - Hierarchy comes from rule weight and spacing, not shadows. Exactly one shadow exists in the app, on the element detail card.
> - Never use `eval()` or `new Function()` for the graphing tool or the molar calculator. Write a tokeniser and a shunting-yard parser.
> - The equation balancer uses Gaussian elimination over rationals on the element-count matrix. Not brute force.
> - All numeric data uses `font-variant-numeric: tabular-nums`.
> - `prefers-reduced-motion` disables all non-essential animation. Focus rings are visible and never removed.
> - Every panel and sub-panel is hash-routable and bookmarkable.
> - Lazy-init each tab on first activation. First paint under 400 ms.
> - Where you generate data I didn't supply (the `EXT` element fields), set anything you're not confident about to `null` rather than guessing, and list those gaps for me at the end of the phase.
>
> Start with Phase 1 only. Before writing code, give me your one-paragraph plan for the shell and confirm the token names you'll use.

**Follow-up prompts, one per phase:**

- *Phase 2:* "Phase 1 works. Build Phase 2 — the periodic table, complete. Generate the full `EXT` dataset for all 118 elements first as a separate step so I can spot-check it, then build the UI on top."
- *Phase 3:* "Build the chemistry tab. Start with the molar calculator and equation balancer. Write test cases first: molar mass of `Ca(OH)2` = 74.09, `CuSO4.5H2O` = 249.68, and balancing `Fe + O2 -> Fe2O3` gives 4, 3, 2. Show me the tests passing before the UI."
- *Phase 4:* "Build the physics tab per §4 Tab 3. The sig-fig and uncertainty tool needs to handle propagation for +, −, ×, ÷ and powers correctly — show me your worked logic before coding it."
- *Phase 5:* "Build the maths tab. The expression parser is the risky part — build and test it standalone against `2x^2-3x+1`, `sin(x)/x`, `e^(-x^2)` before wiring it to the canvas."
- *Polish:* "Full pass: check every tab at 375 px, 768 px and 1440 px; check dark mode contrast on every surface; check tab-through order; check print output for the periodic table and each formula sheet; report the file size."

---

## 9. Acceptance checklist

Run this before you call it done.

- [ ] File opens from `file://` with no console errors and no network requests.
- [ ] Works with wifi off, including fonts falling back cleanly.
- [ ] Under 500 KB.
- [ ] All 118 elements present, correct groups and periods, table shape correct including the f-block offset.
- [ ] No `undefined` or `NaN` visible anywhere — gaps render as "—".
- [ ] Trend overlays produce a sensible gradient across the table (radius decreases left→right, increases top→bottom; if yours doesn't, your data is wrong).
- [ ] Molar mass matches a calculator for five test compounds including a hydrate and a nested bracket.
- [ ] Balancer handles `C3H8 + O2 -> CO2 + H2O` and refuses gracefully on an impossible equation.
- [ ] Every tab reachable by keyboard alone; focus visible at every stop.
- [ ] Dark mode: no surface below 4.5:1 for body text.
- [ ] Refresh restores the last route and theme.
- [ ] Periodic table prints to one A4 landscape page, legible.
- [ ] 375 px wide: nothing overflows horizontally except the deliberate wide-table scroll.

---

## 10. Known traps

1. **The grid gap trap.** `grid-template-columns: repeat(18,1fr)` with a gap and `aspect-ratio:1` will overflow on narrow screens. Set `min-width:0` on tiles and wrap the grid in a container with `overflow-x:auto`.
2. **Lanthanide offset.** The f-block rows sit under the main table, offset to start at column 3. Place them in a separate grid with `grid-column-start:3`, not by faking empty cells.
3. **Search that removes tiles** destroys the table's shape. Dim, never hide.
4. **Hydrate parsing.** `CuSO4.5H2O` — the dot is a separator, not a decimal point. Handle it before tokenising.
5. **Sulfur/aluminium spelling.** NZ uses "sulfur" (IUPAC) and "aluminium". Get this right; it's what your teacher marks.
6. **Rounding in the molar calculator.** Show the full sum and round only at display. Rounding each element's subtotal first gives answers that don't match the marking schedule.
7. **`localStorage` in a `file://` page** works in Chrome but is origin-scoped to the file path. If you move the file, the data doesn't follow. Add a JSON export/import button so your flashcards and credit tracker survive.
8. **Scope creep back into "everything."** Every time you're tempted to add a tab, ask whether you've opened the last one in the past week.

---

## 11. If you want to cut this down

The 80/20 version, buildable in one sitting: the shell, the periodic table with trends, the molar calculator, and the physics constants table. That is genuinely the four things you'll use most. Everything else in this document is expansion room, not a requirement.
