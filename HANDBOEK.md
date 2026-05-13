# Drank Eiland — Developer Handbook

**Doel van dit document:** Een toekomstige Claude (of mens) moet hieruit binnen 5 minuten begrijpen hoe het spel werkt, en daarna een nieuwe minigame kunnen toevoegen zonder bestaande mechanics te breken. Lees dit volledig vóór je code wijzigt.

---

## 1. Wat is dit spel?

Drank Eiland is een mobiel (iPhone-first) 3D bordspel-drinkspel, geïnspireerd op de **Wii Party "Board Game Island"** modus. Spelers gooien om de beurt een dobbelsteen op een fysieke tafel en bewegen langs een pad omhoog op een eiland richting een gouden top. Onderweg landen ze op vakjes met verschillende effecten: vooruit/achteruit-stappen, slokken drinken, minigames triggeren, of catastrofes zoals in de vulkaan vallen.

**Kernfilosofie:**

- **Pass-the-phone**: één telefoon wordt rondgegeven, dus alle eind-resultaat pop-ups vereisen een **bewuste tik** om door te gaan. Niets verdwijnt automatisch.
- **Nostalgie**: chunky cartoon-stijl, "POP!" stempel-animaties, vulkaan-uitbarstingen, kleurrijke tegels.
- **Drinkspel-eerst**: elke actie heeft een slok-implicatie, schaalbaar per moeilijkheidsgraad.
- **iPhone safe-areas overal respecteren** (notch + home indicator).

---

## 2. Bestandsstructuur

```
index.html          ← hoofdspel (Three.js scene, alle game-logica, UI, minigames)
dice-overlay.html   ← Aurum-dice iframe (fysica dobbelstenen, transparante tafel)
raftwar.html        ← Raft Battle minigame (Three.js + physics)
HANDBOEK.md         ← dit document
```

Alle drie de HTML-bestanden moeten in dezelfde folder op GitHub Pages staan. Geen build-step, geen package.json. Three.js en cannon-es worden via CDN geladen (importmap in elke file).

**Waarom losse iframes voor sommige minigames?**

- Three.js / Cannon scenes botsen niet (geen globale namespace-conflicten)
- Geheugen wordt automatisch vrijgegeven als de iframe verdwijnt
- Crash-isolatie: een minigame-bug kraakt het hoofdspel niet
- Lazy-loading: alleen geladen wanneer de minigame wordt gespeeld

**Wanneer inline canvas / DOM?** Voor 2D-canvas minigames (Tappy Bird, Paddle Duel) of pure DOM-overlays (Chwazi, Krokodil, Mystery Wheel, Vulkaan Roulette, Blackjack van de Goden, Hoofdsteden Vuurdoop). Geen aparte scene → geen iframe nodig.

---

## 3. Spelregels (Game Rules)

### Speloverzicht

1. Setup-scherm: spelers voeren namen + kleuren in (2–8 spelers), kiezen een moeilijkheid (Gezellig / Standaard / Pittig / Berserk), en starten het spel.
2. **Tegel 0 (START)** → speler gooit dobbelsteen → token loopt N tegels vooruit.
3. **Landing-effect** wordt uitgevoerd op basis van het tile-type.
4. **Volgende speler** is aan de beurt (tenzij doubles → nog eens).
5. Eerste speler op de **gouden summit (tegel 55)** wint. Iedereen anders drinkt.

### Dobbelsteen-mechanica

- Standaard dobbelsteen: 1–6, iedereen heeft altijd één.
- **Bonus gouden dobbelsteen**: 1–6, gewonnen via minigames. Wordt naast de witte gegooid.
- **Doubles**: als de witte en gouden dobbelsteen dezelfde waarde tonen → **NOG EEN BEURT** (zelfde speler gooit opnieuw).
- **Vulkaan-uitbarsting actief**: −1 op de worp (minimum 1 stap).

### Beurtvolgorde

Spelers gaan in de volgorde waarin ze in de setup zijn toegevoegd. Geen mini-game voor turn-order zoals in Wii Party — we houden het simpel.

### Speciale states per speler

- `skipNextTurn`: speler slaat één beurt over (bv. na in de vulkaan vallen).
- `bonusDice`: aantal gouden dobbelstenen op voorraad.
- `extraRoll`: gooit nog een keer (door doubles).

### Vulkaan-uitbarsting

- Triggered door op een Skull-vakje te landen (50% kans) of op Lava (15% kans).
- Duurt 3 ronden. Banner verschijnt bovenaan.
- Effect: −1 op alle dobbelsteen-worpen, lava-vakjes worden +3 tegels knockback in plaats van glide.

---

## 4. De `G` global game state

```js
const G = {
    players: [],          // array van player-objecten (zie hieronder)
    turnIdx: 0,           // index in G.players
    round: 1,             // ronde-teller (verhoogt elke keer turnIdx wrapt naar 0)
    phase: 'setup',       // 'setup' | 'idle' | 'rolling' | 'moving' | 'event' | 'over'
    volcanoActive: 0,     // ronden over van uitbarsting
    extraRollPending: false,
    difficulty: 'normal', // 'gezellig' | 'normal' | 'pittig' | 'berserk'
};
```

### Player object schema

```js
{
    name:        'Speler 1',
    colorIdx:    0,                    // index in PLAYER_COLORS
    color:       { name, hex, css },
    tileIdx:     0,                    // 0..TILE_COUNT-1
    token:       <THREE.Group>,        // 3D pawn (alleen tijdens een board game)
    bonusDice:   0,                    // gouden dobbelstenen op voorraad
    extraRoll:   false,                // true na doubles → gooit weer
    skipNextTurn: false,               // true na skull → slaat beurt over
}
```

### Phase-flow

```
setup  → idle (game starts)
idle   → rolling (dice button clicked)
rolling → moving (dice settled, token starts walking)
moving → event (token landed, effect triggered)
event  → idle (effect done, turn advances)
event  → over (winner reached summit)
```

**Belangrijk:** `phase` MOET correct beheerd worden of de dobbel-knop doet niets meer. Iedere event-functie moet eindigen in `idle` of de turn-advance moet expliciet plaatsvinden.

---

## 5. Het tegel-systeem

### Bord layout

- 56 tegels totaal (`NUM_TILES`)
- Tegel 0 = START (groen)
- Tegel 50–54 = LAVA zone (oranje)
- Tegel 55 = GOUDEN SUMMIT (winst)
- Pad gevolgd door `pathCurve` (CatmullRomCurve3 over `pathPoints`)

### Hoe wordt een tegel-type bepaald?

```js
function tileTypeFor(i) {
    if (i === 0) return 'start';
    if (i >= NUM_TILES - 6 && i < NUM_TILES - 1) return 'lava';
    if (SPECIAL_TILES[i]) return SPECIAL_TILES[i];  // vaste posities
    // Chwazi blijft een aparte tegel — eigen mechanic (random drinker).
    if (i > 0 && i % 9 === 0) return 'chwazi';
    // ⚡ NIEUW: alle andere minigame-tegels (vroeger krokodil/paddle/tappy/volcanoHop)
    // worden samengetrokken tot één 'mystery' tegel die een rad spint.
    if (i > 0 && (i % 13 === 0 || i % 11 === 0 || i % 7 === 0 || i % 17 === 0))
        return 'mystery';
    // Filler: blauw/rood/groen op basis van een vaste hash
    const r = Math.sin(i * 9.13) * 0.5 + 0.5;
    if (r < 0.40) return 'plus';
    if (r < 0.72) return 'minus';
    return 'drink';
}
```

`SPECIAL_TILES` is een hardcoded dict met de zeldzame speciale vakjes:

```js
{ 15: 'bluestatue', 19: 'tornado', 23: 'ufo',
  30: 'skull', 32: 'bluestatue', 37: 'tornado',
  43: 'ufo', 47: 'redstatue' }
```

### Alle tile-types

| Type | Visueel | Effect | Standaard slokken |
|---|---|---|---|
| `start` | Groen | Begin | — |
| `plus` | Blauw + drijvende bol | +2 vooruit | drink 1 (lucky tax) |
| `minus` | Rood + drijvende bol | −2 achteruit | drink 2 |
| `drink` | Groen + drijvende bol | Cheers met gekozen speler | beiden 1 |
| `bluestatue` | Glow-zuil blauw | +6 mega-boost | 1 |
| `redstatue` | Glow-zuil rood + screen-shake | −6 knockback | 3 |
| `tornado` | Roterende kegel | Kies slachtoffer, −3 + drink 2 | victim 2 |
| `ufo` | Drijvende schotel | Swap met gekozen speler | beiden 1 |
| `skull` | Schedel + zwart vakje | In de vulkaan: drink 4, skip 1 beurt, 50% eruption | 4 |
| `lava` | Oranje gloeiend | Drink 1, kans op slip & eruption | 1–2 |
| `chwazi` | Geel | Random player drinkt | 3 |
| **`mystery`** | **Felle magenta (rgb#ff3acf)** | **Spint Mystery Wheel → random minigame** | **varieert** |

**Legacy types** (niet meer gegenereerd door `tileTypeFor`, maar wel ondersteund in `onLandedOnTile` voor back-compat):
`krokodil`, `paddle`, `tappy`, `volcanoHop`.

### Beweging (forwards & backwards)

Drie hulpfuncties bestaan al — gebruik deze, niet hardcoded:

```js
moveTokenAlong(player, steps, onDone)   // VOORWAARTS met camera-follow + tile events
moveTokenBy(player, deltaTiles, onDone) // BEIDE richtingen, GEEN tile events (voor knockback)
teleportTokenTo(player, tileIdx)        // Instant zonder animatie (voor UFO swap)
```

**Belangrijke regel uit Wii Party:** Als je een speler verplaatst door een tile-effect, mag de NIEUWE landing GEEN nieuw event triggeren. `moveTokenBy()` skipt events. Alleen de win-check moet je handmatig doen na de move.

---

## 6. Het difficulty-systeem

```js
const DIFFICULTY_CONFIGS = {
    gezellig: { drinkMult: 0.5, knockMult: 0.7, skullEnabled: false,
                redStatueEnabled: false, eruptionChance: 0.1, startBonusDice: 1 },
    normal:   { drinkMult: 1,   knockMult: 1,   skullEnabled: true,
                redStatueEnabled: true,  eruptionChance: 0.2, startBonusDice: 0 },
    pittig:   { drinkMult: 1.5, knockMult: 1.3, /* ... */ },
    berserk:  { drinkMult: 2.2, knockMult: 1.5, /* ... */ },
};
```

**Twee verplichte helpers** in elke nieuwe tile/minigame:

```js
slokken(base)   // pas drinkMult toe, min 1. Voorbeeld: slokken(4) → 6 op Pittig
knockBy(base)   // pas knockMult toe, min 1. Voorbeeld: knockBy(3) → 4 op Berserk
```

**GEBRUIK ALTIJD DEZE HELPERS** voor drink/knockback waarden. Hardcoded `drinks: 4` in een nieuwe minigame is een bug — gebruik `drinks: slokken(4)`.

### Speciale gedragingen per moeilijkheid

- **Gezellig**: skull wordt "BRRR…" event (alleen drink 2), redstatue wordt mild (−2 i.p.v. −6). Iedereen start met 1 gouden dobbelsteen.
- **Pittig & Berserk**: hogere eruption-kans, meer slokken.

---

## 7. Dobbelsteen-systeem (Aurum iframe)

### postMessage protocol

`index.html` laadt `dice-overlay.html` als iframe per worp. Communicatie:

```
iframe → host:  'dice-ready'      // klaar voor instructies
iframe → host:  'dice-result'     // { total, values:[v1,v2], doubles }
iframe → host:  'dice-dismiss'    // gebruiker tikte Verder

host  → iframe: 'dice-init'       // { count, hasGoldDie, playerName }
host  → iframe: 'dice-roll'       // start de fysica-worp
```

### Flow van één worp

1. Host opent iframe → wacht op `dice-ready`
2. Host stuurt `dice-init` met aantal dobbelstenen + of er een gouden bij hoort
3. Na 350ms → host stuurt `dice-roll`
4. Fysica draait → na settle (0.35s stilte) → iframe stuurt `dice-result`
5. Verder-knop verschijnt in iframe
6. Speler tikt → iframe stuurt `dice-dismiss`
7. Host verwijdert iframe, beweegt de token

### Eigenschappen van de tafel

- **Tafel = `THREE.CircleGeometry(5.3)`** ([was 6.2 vóór de iPhone-fix](#krokodil-en-tafel-fix-mei-2026)) — opacity 0.55 zodat het eiland erdoorheen schemert
- Goud-inlay ring: radius 4.95–5.05 (was 5.85–5.95)
- Tweede dunne inner-ring: 4.70–4.73 (was 5.55–5.58)
- Outer ring + corner ornaments uitgeschakeld in overlay-mode
- Body achtergrond fully transparent
- Scene-fog uit
- **Fysica-walls: `WALL_RADIUS = 4.65`** (was 5.5) — zo blijven de dice binnen de zichtbare tafel op een iPhone SE / 13 mini

### Bonus-dobbelsteen

- Wanneer `state.useGold === true`: dobbelsteen index 0 = ivoor wit, index 1 = goud (gerenderd met aurum gold-palette)
- `pickMaterialFor(i)` regelt dit

---

## 8. UI-bouwstenen

Vier reusable primitieven, gebruik ze altijd:

### `showModal({ emoji, title, body, actions })`

Klassiek modaal met knoppen onderaan. Returnt Promise met de gekozen value.

```js
const go = await showModal({
    emoji: '🐊', title: 'Begin?',
    body: ['Tekst regel 1', 'Tekst regel 2'],
    actions: [{ label: 'Start', value: true }, { label: 'Skip', value: false }],
});
```

### `showStamp({ emoji, big, sub, palette, auto, duration })`

"Pop!" stempel-animatie. **Standaard vereist een tik om door te gaan.** Set `auto: true` voor korte transities die direct gevolgd worden door een ander interactie-element.

```js
// EIND-resultaat: speler tikt Verder
await showStamp({ emoji: '🎲', big: '+2!', sub: 'Drink 1', palette: 'blue' });

// LEAD-IN naar pickVictim: auto-dismiss
await showStamp({ emoji: '🌪️', big: 'TORNADO!', sub: 'Kies...',
                  palette: 'dark', auto: true, duration: 1300 });
```

Palettes: `gold`, `green`, `red`, `blue`, `purple`, `dark`, `lava`.

### `pickVictim({ title, subtitle, exclude })`

Toont een lijst met spelerknoppen om een doelwit te kiezen. Returnt het gekozen player-object (of `null` bij geen kandidaten).

```js
const victim = await pickVictim({
    title: '🌪️ Wie wordt weggeblazen?',
    subtitle: `−${n} tegels achteruit + drink ${d} 🍻`,
    exclude: [player],  // de huidige speler kan zichzelf niet kiezen
});
```

### `screenShake(ms)`

Voegt de `.shake` class toe aan het canvas. Gebruik voor harde knockbacks.

### `playChwazi({ title, subtitle, drinks })`

Vinger-op-scherm random picker. Returnt de gekozen speler. Goed voor "iemand drinkt"-events.

---

## 9. Bestaande minigame-patronen

Bestudeer deze vier voorbeelden — alle nieuwe minigames passen in een van deze categorieën.

### Pattern A: 1v1 inline canvas (DOM overlay)

Voorbeeld: `playPaddleDuel`, `playKrokodil`, `playGodBlackjack`, `playHoofdsteden`, `playVulcanoRoulette`, `playMysteryWheel`

- Full-screen `<div>` overlay met `position: fixed; inset: 0; z-index: 60`
- Canvas of DOM-elementen erin
- Touch handlers via `touchstart`/`move`/`end`, of klassieke `click` voor buttons
- Eind: knop "Verder" → resolve Promise met `loser` / `winner` / payload-object

**Wanneer gebruiken:** simpele 2D physics of tap-events, 1–2 spelers, snel klaar.

Boilerplate:

```js
function playMijnMinigame({ playerA, playerB, drinks = 4 } = {}) {
    return new Promise(resolve => {
        const ui = document.getElementById('game-ui');
        const back = el('div', { className: 'mm-back' });
        // ... build UI ...
        ui.append(back);
        function finish(loser) {
            back.style.transition = 'opacity 0.3s ease';
            back.style.opacity = '0';
            setTimeout(() => { back.remove(); resolve(loser); }, 300);
        }
    });
}
```

### Pattern B: 1v1 iframe (eigen Three.js scene)

Voorbeeld: `playRaftBattle` → `raftwar.html`

- Iframe-overlay zoals dice-overlay
- postMessage protocol:
  - iframe stuurt `raft-hello` → host stuurt config
  - iframe stuurt `raft-ready` → host weet dat config aankwam
  - iframe stuurt `raft-knockout` met `winner` → host toont KO-screen
- Manual fallback-knoppen ("Wie won?") voor als de iframe niet reageert

**Wanneer gebruiken:** complexe Three.js scene, eigen physics, alles wat in het hoofd-scene namespace-conflicten zou geven.

### Pattern C: All-players inline (pass-the-phone)

Voorbeeld: `playTappyBird`, `playVolcanoHop`

- Elke speler speelt om de beurt op dezelfde telefoon
- Tussenscherm: "Geef de telefoon aan Speler X"
- Eindscoreschermen vergelijken alle spelers
- Winnaar krijgt typisch een bonus-dobbelsteen

**Wanneer gebruiken:** solo skill-test waar alle spelers eerlijke kans willen krijgen.

### Pattern D: Random / Chwazi-style

Voorbeeld: `playChwazi`

- Multi-touch finger detection
- Random pick met dramatische selectie
- 1 winnaar (of verliezer) krijgt het drink-effect

**Wanneer gebruiken:** "iemand willekeurig drinkt" events.

---

## 10. ⚡ NIEUW: De Mystery Wheel & universele minigame-tegel

Sinds de laatste update worden de individuele minigame-tegels (krokodil/paddle/tappy/volcanoHop) samengetrokken tot één gemeenschappelijke **`mystery`** tegel. Chwazi blijft zijn eigen ding.

### Hoe werkt het op het bord?

1. Speler landt op een `mystery` tegel.
2. `showStamp` met "MYSTERY!" speelt af (auto-dismiss, 1.2s).
3. `playMysteryWheel({ options })` opent — een full-screen radspinner met de beschikbare minigames.
4. Speler tikt "🎡 Draai!" → wiel spint 4.5s en stopt op een random optie.
5. "▶️ Start!" knop verschijnt → de gekozen minigame opent via `runMinigameOnTile(id, player)`.

### `MYSTERY_OPTIONS` array

```js
const MYSTERY_OPTIONS = [
    { id: 'krokodil',    emoji: '🐊',  label: 'Krokodil',     color: '#b87aff' },
    { id: 'paddle',      emoji: '🏓',  label: 'Paddle Duel',  color: '#4af2ff' },
    { id: 'tappy',       emoji: '🐦',  label: 'Tappy Bird',   color: '#ff7ad6' },
    { id: 'volcano',     emoji: '🌋',  label: 'Volcano Hop',  color: '#3aff9e' },
    { id: 'raft',        emoji: '🏴‍☠️', label: 'Raft Battle',  color: '#ff8a4a' },
    { id: 'roulette',    emoji: '🎰',  label: 'Roulette',     color: '#ff3a3a' },
    { id: 'blackjack',   emoji: '🃏',  label: 'Blackjack',    color: '#2a8a3a' },
    { id: 'hoofdsteden', emoji: '🏙️',  label: 'Hoofdsteden',  color: '#5aaaff' },
    { id: 'bowling',     emoji: '🎳',  label: 'Bowlspel',     color: '#1a7adb' },
    { id: 'connect4',    emoji: '🟡',  label: '4 op een Rij', color: '#ffcc1a' },
];
```

Wil je een nieuwe minigame in het wiel zetten? Voeg een entry toe aan `MYSTERY_OPTIONS` + een `else if (id === 'jou-id')` block in `runMinigameOnTile(id, player)`.

### `runMinigameOnTile(id, player)` — de centrale dispatcher

Deze helper draait de juiste minigame inclusief opponent-keuze, drink-schaling, bonus-dobbelsteen reward en het eind-resultaat modaal. Wordt aangeroepen door:

- De `'mystery'` tegel (na het wiel)
- De legacy minigame-tegels (`krokodil`, `paddle`, `tappy`, `volcanoHop`) voor back-compat
- **Niet** door de standalone home-screen launcher — die heeft een eigen pad zonder bord-movement (zie §12).

---

## 11. ⚡ NIEUW: De drie extra minigames

### 11.1 Vulkaan Roulette (`playVulcanoRoulette`)

**Pattern A • solo • CSS-prefix `vr-`**

- **Fase 1**: speler tikt om virtuele 1–6 dobbelsteen te gooien (de **inzet**).
- **Fase 2**: speler kiest 🔴 Rood, ⚫ Zwart, of 🟡 Goud (×3 payout).
- **Fase 3**: 16-segment wiel spint (7 rood, 7 zwart, 2 goud → ~12.5% goud-kans).
- **Uitkomst**:
  - Rood/zwart correct → +`bet` tegels vooruit, drink `slokken(1)` (lucky tax)
  - Goud correct → +`bet × 3` tegels vooruit + **bonus-dobbelsteen** (jackpot)
  - Fout → −`bet` tegels achteruit, drink `slokken(bet)` 🍻

Return value: `{ won, color, pick, bet, steps, drinks }` (of `null` bij skip).

### 11.2 Blackjack van de Goden (`playGodBlackjack`)

**Pattern A • 1 speler vs CPU • CSS-prefix `bj-`**

- Standaard 21-regels: kaarten 2–10 hun waarde, J/Q/K = 10, **Aas = 1 of 11 (auto-best)**.
- Speler krijgt 2 kaarten, dealer ('Vulkaan God') 2 (één gedekt).
- Hit / Stand knoppen.
- Dealer-AI: hit tot ≥ 17 (stand op soft 17).
- **Uitkomst**:
  - **Winst**: speler kiest een slachtoffer via `pickVictim` → die drinkt `slokken(4)` + `knockBy(3)` tegels achteruit. Winnaar krijgt ook een gouden bonus-dobbelsteen.
  - **Verlies**: huidige speler drinkt zelf `slokken(4)` + `knockBy(3)` achteruit.
  - **Gelijk**: niets gebeurt.

Return value: `{ won, tie, playerVal, dealerVal, drinks }` (of `null` bij skip).

### 11.3 Hoofdsteden Vuurdoop (`playHoofdsteden`)

**Pattern A • solo • CSS-prefix `hf-`**

- Een willekeurig land uit de `CAPITALS` array verschijnt.
- 12 seconden timer (groen → geel → rood gradient bar).
- Speler typt het antwoord en drukt Enter (of de ▶ knop).
- **String-vergelijking**: case-insensitive, accent-normalised (NFD + diacritic strip), trimmed, multi-whitespace genormaliseerd. Per land staat een array van **acceptabele antwoorden** (bv. Sri Lanka accepteert zowel "Colombo" als "Sri Jayawardenepura Kotte").
- **Uitkomst**:
  - **Correct & op tijd**: +`knockBy(3)` tegels vooruit + **bonus-dobbelsteen** + drink `slokken(1)` lucky tax
  - **Fout / te laat**: −`knockBy(2)` tegels achteruit + drink `slokken(2)`

**Standaard `CAPITALS` lijst (20 items, allemaal uitdagend):**
Mongolië · Kazachstan · Bhutan · Eritrea · Vanuatu · Sri Lanka · Suriname · Burkina Faso · Madagaskar · Kirgizië · Botswana · Brunei · Tadzjikistan · Oezbekistan · Azerbeidzjan · Tsjaad · Djibouti · Lesotho · Bahrein · Litouwen.

Wil je hem makkelijker? Voeg gewone landen toe aan de array. Wil je hem moeilijker? Verkort de `timeMs` parameter (default 12000).

Return value: `{ won, correct, timeout, country, expected, given, drinks }` (of `null` bij skip).

---

## 12. Een nieuwe minigame toevoegen — stap-voor-stap checklist

### Stap 1 — Pattern kiezen

Vraag: hoeveel spelers? Eigen physics nodig?

- 1 speler, simpel 2D → **Pattern A (inline)**
- 2 spelers, simpel 2D → **Pattern A (inline)**
- 2 spelers, complex 3D → **Pattern B (iframe)**
- Alle spelers, individueel → **Pattern C (pass-the-phone)**
- 1 random verliezer → **Pattern D (Chwazi-clone)**

### Stap 2 — Mystery Wheel of eigen tegel?

- **Universeel (aanbevolen)**: voeg een entry toe aan `MYSTERY_OPTIONS` + `runMinigameOnTile()`. Zo wordt de minigame een mogelijke uitkomst van het rad. Geen tile-type wijziging nodig.
- **Eigen tegel**: alleen als de minigame een sterk eigen identiteit heeft (zoals Chwazi):
  1. Kies een naam, bv. `'sumoring'`
  2. Voeg toe aan `TILE_MATS` met een unieke kleur
  3. Voeg toe aan `TILE_TYPE_TO_MAT` mapping
  4. Voeg trigger toe in `tileTypeFor()` (procedural of via `SPECIAL_TILES`)
  5. Voeg een `else if` block toe in `onLandedOnTile`

### Stap 3 — CSS toevoegen

Boven aan de `<style>` blok in `index.html`, naast bestaande minigame-CSS. Prefix alle classes met een eigen 2-letter code:

| Prefix | Minigame |
|---|---|
| `pd-` | Paddle Duel |
| `tb-` | Tappy Bird |
| `vh-` | Volcano Hop |
| `kroko-` | Krokodil |
| `mw-` | Mystery Wheel |
| `vr-` | Vulkaan Roulette |
| `bj-` | Blackjack |
| `hf-` | Hoofdsteden |

```css
.mm-back { position: fixed; inset: 0; z-index: 60;
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
  /* ... */ }
```

**LET OP iPhone safe-areas**: gebruik `env(safe-area-inset-top/bottom)` op de root-container, OF zet content op minimaal 70px van de rand. Voor minigames met knoppen onderaan: zet de safe-area-bottom op de **footer**, niet op de back-container, anders krijg je een zwarte rand die de knoppen onbereikbaar maakt (zie Krokodil-fix in §15).

### Stap 4 — `playMijnMinigame()` functie schrijven

Volg de boilerplate uit Pattern A/B/C/D. Belangrijke regels:

- Functie returnt een Promise met de loser/winner/payload/null
- Gebruik `drinks: slokken(N)` in plaats van hardcoded waarden
- Eind-overlay heeft altijd een "Verder" knop die de promise resolved
- Skip-knop optioneel maar aanbevolen voor testing

### Stap 5 — In Mystery Wheel registreren

Voeg toe aan `MYSTERY_OPTIONS`:

```js
{ id: 'sumo', emoji: '🥋', label: 'Sumo Ring', color: '#ff7a00' },
```

En een handler in `runMinigameOnTile(id, player)`:

```js
else if (id === 'sumo' && G.players.length >= 2) {
    const opp = pickOpp();
    const d = slokken(4);
    const go = await showModal({ /* intro */ });
    if (!go) return;
    const loser = await playSumoRing({ playerA: player, playerB: opp, drinks: d });
    if (loser) {
        const winner = loser === player ? opp : player;
        winner.bonusDice = (winner.bonusDice || 0) + 1;
        refreshHUD();
        await showModal({ /* resultaat */ });
    }
}
```

### Stap 6 — Standalone-tab entry

In `renderSetup()`, vind de `MINIGAMES` array en voeg een entry toe:

```js
{ id: 'sumo', emoji: '🥋', name: 'Sumo Ring', meta: '1v1', tag: 'NIEUW', locked: false },
```

Daarna in `launchStandaloneMinigame()` handle de id:

```js
} else if (id === 'sumo' && G.players.length >= 2) {
    const a = G.players[0], b = G.players[1];
    const loser = await playSumoRing({ playerA: a, playerB: b, drinks: 4 });
    if (loser) {
        const winner = loser === a ? b : a;
        winner.bonusDice = (winner.bonusDice || 0) + 1;
        refreshHUD();
    }
}
```

### Stap 7 — Bonus-dobbelsteen belonen

**Verplicht voor competitive minigames** (1v1 of "wie wint krijgt"):

```js
winner.bonusDice = (winner.bonusDice || 0) + 1;
refreshHUD();
```

Voor multi-player minigames (Volcano Hop, Tappy Bird) krijgt alleen de #1 winnaar er één. Voor solo-skill games (Hoofdsteden, Blackjack-winst) krijgt de speler ook één.

### Stap 8 — Test-checklist

- [ ] Werkt het op een iPhone (touch + safe-area)?
- [ ] Sluiten alle pop-ups via een bewuste tik (geen auto-dismiss)?
- [ ] Schaalt het slokken-aantal met `slokken()`?
- [ ] Krijgt de winnaar een bonus-dobbelsteen?
- [ ] Werkt de Skip-knop?
- [ ] Wordt `G.phase` netjes hersteld naar `idle` na afloop?
- [ ] Test in alle 4 moeilijkheden (Gezellig drink-1 mag niet 0 worden)
- [ ] Test met 2, 4, en 8 spelers
- [ ] Bij toevoeging aan Mystery Wheel: draait het rad correct + komt jouw segment voor?

---

## 13. Veelgemaakte fouten

| ❌ Fout | ✅ Goed |
|---|---|
| `drinks: 4` hardcoded | `drinks: slokken(4)` — schaalt mee met difficulty |
| Overlay ligt over de iPhone-notch | `padding-top: env(safe-area-inset-top); padding-bottom: env(safe-area-inset-bottom);` op root, OF minimaal 70px van de rand |
| Stempel verdwijnt automatisch | Verwijder `duration:` uit `showStamp` — zonder die parameter wacht hij op een tik |
| Winnaar krijgt geen bonus | `winner.bonusDice++; refreshHUD();` na elke 1v1 winst |
| Bottom buttons onbereikbaar (Krokodil-fout) | Zet `env(safe-area-inset-bottom)` op de footer, niet op de back-container. Voeg `flex-shrink: 0` aan de footer toe. |
| Dice rolt over de tafelrand op iPhone | Hou `WALL_RADIUS` ≤ felt-radius − 0.3 (zie §7) |
| G.phase blijft op 'event' hangen | Iedere event-functie moet eindigen in `idle` of `advanceTurn()` moet expliciet aangeroepen worden |
| Twee minigames triggeren tegelijk | De `else if`-keten in `onLandedOnTile` is exclusief — voeg altijd toe met `else if`, nooit `if` |
| Recursieve event-trigger door knockback | `moveTokenBy()` doet NOOIT events. Alleen `moveTokenAlong()` triggert events op landing |
| Tile decoration vergeten | Speciale tegels horen 3D-decoraties (zuilen/orbs) te hebben in `// Special tile decorations` |

### Z-index hiërarchie

Hou je hieraan, anders verdwijnt iets ineens achter een ander overlay:

```
10 = #ui (HUD)
20 = #top-bar, vol-banner
50 = .modal-back
55 = .tile-stamp-back
56 = .vp-back (victim picker)
60 = minigame overlays (.pd-back, .kroko-back, .mw-back, .vr-back, .bj-back, .hf-back, etc)
70 = .dice-iframe-back
```

---

## 14. Codebase quick-reference

### Belangrijke globals

```
NUM_TILES         = 56
TILE_COUNT        = NUM_TILES
PLAYER_COLORS     = [{name, hex, css}, ...] (8 kleuren)
SPECIAL_TILES     = { 15:'bluestatue', ... }
DIFFICULTY_CONFIGS = { gezellig, normal, pittig, berserk }
G                 = de game-state singleton
tiles             = array van { mesh, baseY, phase, bobMesh, type }
tileTypes         = array van strings ('plus', 'minus', 'mystery', ...)
tileWorldPos      = array van THREE.Vector3 (positie van elke tegel)
pathCurve         = CatmullRomCurve3
MYSTERY_OPTIONS   = array van { id, emoji, label, color } voor de wheel
CAPITALS          = array van { country, answers[] } voor Hoofdsteden
```

### Belangrijke functies — alfabetisch

```
advanceTurn()                  → naar volgende speler, ronde-counter, volcano-decrement
goHome()                       → reset alles, terug naar setup-scherm
knockBy(base)                  → diff-aware knockback distance
layoutTokensOnTile(i)          → tokens netjes uitwaaieren op tegel i
moveTokenAlong(p, n, cb)       → vooruit MET events
moveTokenBy(p, ±n, cb)         → beide richtingen ZONDER events
onLandedOnTile(p)              → dispatcher voor alle tile-effecten
onRollDice()                   → start de dobbel-iframe
pickVictim({...})              → speler-kiezer modal
playChwazi(...)                → vinger-op-scherm picker
playGodBlackjack(...)          → ⚡ NIEUW: 21 vs CPU
playHoofdsteden(...)           → ⚡ NIEUW: trivia onder tijdsdruk
playKrokodil(...)              → tap-on-tooth picker
playMysteryWheel(...)          → ⚡ NIEUW: rad dat een minigame kiest
playPaddleDuel(...)            → pong
playRaftBattle(...)            → iframe-based knockout
playTappyBird(...)             → flappy bird pass-the-phone
playVolcanoHop(...)            → kies-platform pass-the-phone
playVulcanoRoulette(...)       → ⚡ NIEUW: rood/zwart/goud gokken
refreshHUD()                   → herteken bottom HUD + minimap
renderSetup()                  → setup-scherm met diff-keuze + minigame tab
runMinigameOnTile(id, p)       → ⚡ NIEUW: dispatcher voor mystery + legacy tiles
screenShake(ms)                → canvas shake
showModal({...})               → klassieke knoppen-modal
showStamp({...})               → Wii Party POP! stempel (klik-vereist standaard)
slokken(base)                  → diff-aware drink count
startGame(draft)               → bouwt spelers, plaatst tokens, start game
teleportTokenTo(p, i)          → instant verplaatsen (voor UFO)
tileTypeFor(i)                 → bepaal type voor tegel-index i
updateVolcanoBanner()          → toon/verberg de uitbarsting-banner
```

### postMessage berichten

**Naar `dice-overlay.html`:**

- `{ type: 'dice-init', count, hasGoldDie, playerName }`
- `{ type: 'dice-roll' }`

**Van `dice-overlay.html`:**

- `{ type: 'dice-ready' }`
- `{ type: 'dice-result', total, values, doubles }`
- `{ type: 'dice-dismiss' }`

**Naar `raftwar.html`:**

- `{ type: 'raft-init', playerA: {name, color}, playerB: {...} }`

**Van `raftwar.html`:**

- `{ type: 'raft-hello' }`
- `{ type: 'raft-ready' }`
- `{ type: 'raft-knockout', winner: 'A'|'B'|1|2, winnerName }`

---

## 15. Changelog — Krokodil en tafel-fix (mei 2026)

Twee gerichte fixes voor iPhone usability:

### Dice-tafel (`dice-overlay.html`)

De tafel was net te groot waardoor dobbelstenen vaak over de rand uit beeld rolden op een iPhone (vooral SE / 13 mini). Door de loop heen meermaals geschaald:

| Element | v1 (origineel) | v2 (eerste fix) | v3 (huidig) |
|---|---|---|---|
| Felt geometry | `CircleGeometry(6.2)` | `CircleGeometry(5.3)` | `CircleGeometry(3.5)` |
| Gold inlay ring | `RingGeometry(5.85, 5.95)` | `RingGeometry(4.95, 5.05)` | `RingGeometry(3.26, 3.34)` |
| Inner ring | `RingGeometry(5.55, 5.58)` | `RingGeometry(4.70, 4.73)` | onveranderd |
| Physics walls | `WALL_RADIUS = 5.5` | `WALL_RADIUS = 4.65` | `WALL_RADIUS = 2.9` |
| Camera | `(0, 5.2, 5.0)` FOV 48 | `(0, 7.2, 7.4)` FOV 50 | `(0, 9.5, 9.8)` FOV 52 |

**Lesson learned over de camera op iPhone**: Three.js's `PerspectiveCamera` FOV is **verticaal**. Op een portrait phone (aspect ≈ 0.5) is de horizontale FOV veel smaller dan de verticale. Bij aspect 0.5 en FOV 50 is de horizontale FOV slechts ~26°. Dus de zichtbare horizontale breedte op de tafel is **veel** smaller dan op landscape. De fix is altijd een combinatie van (a) camera verder weg duwen zodat de horizontale visible width groter wordt, EN (b) WALL_RADIUS verkleinen zodat de fysica de dice constraineert binnen wat de camera ziet.

Formule om te checken of een nieuwe camera-positie werkt op iPhone:
```
distance = sqrt(Y² + Z²)
horizontal_half_fov = atan(tan(FOV/2) * 0.5)   // aspect 0.5 voor portrait
visible_half_width = distance * tan(horizontal_half_fov)
// Vereiste: visible_half_width > WALL_RADIUS  (met ~15% marge)
```

### Krokodil-overlay (`index.html`, CSS-prefix `kroko-`)

De onderste knoppen waren niet bereikbaar omdat de safe-area-inset onderaan de back-container een dikke zwarte rand creëerde + de croc-grafiek volledig naar de bodem doorliep over de footer heen. Fix:

- `.kroko-back` padding-bottom: `max(6px, env(safe-area-inset-bottom))` → `4px` (vast)
- `.kroko-stage` kreeg `min-height: 0` zodat flex-shrink werkt
- `.kroko-croc` geschaald: `width: min(80vw, 340px)` (was `min(86vw, 380px)`), `max-height: 64vh` (was `78vh`), rigide `height: min(72vh, 720px)` vervangen door `height: 100%`
- `.kroko-foot` kreeg de safe-area-padding naar binnen verplaatst + `flex-shrink: 0` zodat de buttons altijd zichtbaar zijn

**Lesson learned voor volgende minigames:** zet `env(safe-area-inset-bottom)` op je footer, niet op de root-back container. Dat voorkomt dat de safe-area de hele scene onderaan donkergrijs maakt en daarmee interactieve elementen onbereikbaar.

### Setup/minigame-scherm scroll-fix

Op een iPhone werd lang content (8 spelers + difficulty + start) afgeknipt omdat `.screen` gebruik maakte van `position: fixed; inset: 0` met `justify-content: center` zonder overflow. Fix:
- Verwijderd `justify-content: center`
- Toegevoegd `overflow-y: auto; -webkit-overflow-scrolling: touch; overscroll-behavior: contain`
- Vervangen door spacer-pseudo-elements `.screen::before, .screen::after { flex: 1 1 0; min-height: 0 }` die de content verticaal centreren als die past, maar inklappen tot 0 als de viewport overflows. Standard pattern voor "verticaal gecentreerd indien past, anders scroll van bovenaf".

### Bowlspel + Vier op een Rij toegevoegd (mei 2026, v2)

Twee minigames toegevoegd als losse iframe-bestanden, exact zoals Raft Battle:

**`bowlspel.html`** — Three.js r128 (global namespace) bowling met 2-10 spelers, één frame per speler. Postmessage protocol:
- iframe → host: `{type: 'bowl-hello'}` (klaar voor init)
- host → iframe: `{type: 'bowl-init', players: [{name, color}, ...]}` (skip setup screen, start direct)
- iframe → host: `{type: 'bowl-result', winner, winnerScore, ranking}` (na showFinalResults)
- host → iframe: `{type: 'bowl-skip'}` (afbreken, antwoord met huidige leider)

**`vier-op-een-rij.html`** — Pure DOM/CSS 1v1 connect-four. Postmessage protocol:
- iframe → host: `{type: 'cf-hello'}`
- host → iframe: `{type: 'cf-init', playerA: {name, color}, playerB: {name, color}}` (override hardcoded Mio/Suni)
- iframe → host: `{type: 'cf-result', winner: 1|2|0, winnerName}` (0 = gelijkspel)
- host → iframe: `{type: 'cf-skip'}` (afbreken)

In `index.html` zijn `playBowling({ players, drinks })` en `playConnectFour({ playerA, playerB, drinks })` toegevoegd, en beide minigames zitten in `MYSTERY_OPTIONS` + `MINIGAMES` (home-screen tab).

### 🥃 Schnaps ruft an (mei 2026, v3)

Een random, **niet-tegel-gebonden** event toegevoegd: tijdens het bordspel kan op willekeurige momenten "Schnaps" jou bellen. iOS-style call-scherm (full-screen blur), 8-bit Nokia 3310 ringtone (Gran Vals, Tárrega 1902, via WebAudio square-wave), met vibratiepatroon op Android.

**De grap**: aannemen = je moet schnapsen (drinken in het Duits), weigeren = "Du bist eine laffe Schnapsaffe!" + tegel-straf. De Duitse termen zijn deel van de joke en blijven Duits.

#### Trigger-logica

- **Geplanned bij `startGame()`**: per difficulty wordt 1–4 absolute turn-nummers uit de range [4, 4+8+N×6) random gekozen. Opgeslagen in `G.schnapsCallTurns` als sorted array.
- **Aantallen per difficulty:**

| Difficulty | Aantal calls |
|---|---|
| Gezellig | 1 |
| Normal | 2 |
| Pittig | 3 |
| Berserk | 4 |

- **Check bij**: einde van `onLandedOnTile()` en einde van skip-turn handler in `onRollDice()`, via `await maybeFireSchnapsCall()`. Alleen wanneer `G.phase === 'idle'` — dus **nooit tijdens een minigame** (die zit op `phase === 'event'`).
- **Target speler**: degene wiens beurt direct gaat beginnen (na `advanceTurn()`).
- **Effect bij aannemen**: `slokken(3)` voor de target speler, met de tekst "Schnapsen, ma!! PROST! {name} trinkt N Schluck".
- **Effect bij weigeren**: `-knockBy(5)` tegels achteruit, screen shake, met tekst "Du bist eine laffe Schnapsaffe!".

#### Home-screen extra

Onder aan het setup-scherm staat een "📞 Schnaps anrufen" knop (vereist scrollen). Daar opent dezelfde Schnaps-call, maar in **standalone modus**: geen straffen, geen result-modal. Nokia ringtone speelt, je tikt opnemen of weigeren, en de animatie verdwijnt. Pure easter-egg.

#### Bestanden

Alles inline in `index.html`:
- CSS-prefix `sk-` voor het call-scherm + result modal
- `playSchnapsCall({ player, mode: 'game'|'standalone' })` — Promise-based, returnt `{accepted, drinks, knockback}` of `{accepted}` voor standalone
- `maybeFireSchnapsCall()` — central check geroepen na elke turn-advance
- `G.schnapsCallTurns` — array van absolute turn-nummers die nog te vuren zijn
- Difficulty config heeft nieuw veld `schnapsCalls: number`

---

## 16. Ideeën voor toekomstige minigames

Aanbevolen om als Mystery Wheel-optie toe te voegen (geen aparte tegel nodig):

| Idee | Pattern | Slokken |
|---|---|---|
| 🎯 Bow Aim (target schieten) | A (inline canvas) | 3 |
| ⚡ Snel-tap reactie | A | 2 |
| 🃏 Higher/Lower kaart-game | A (DOM only) | 3 |
| ✊ Steen-papier-schaar (3 spelers) | A (DOM) | 2 |
| 🪂 Skydiver — wie opent als laatst de parachute | C (pass-the-phone) | 2 |
| 🚣 Roeiwedstrijd (twee duimen) | A (canvas) | 4 |
| 🎲 Liar's Dice mini-versie | A (DOM) | varieert |
| 🐍 Snake (high-score) | C | 2 |
| 🎣 Vissen met timing-bar | A (canvas) | 2 |
| 🔥 Lava-tap (snelheid) | A (canvas) | 3 — trigger tijdens uitbarsting |

Voor elk idee: vraag de gebruiker eerst voor design-keuzes vóór je de minigame bouwt (welke pattern, hoeveel slokken, in het rad of eigen tegel). Niet zomaar bouwen.

---

## 17. Stijl & nostalgie checklist (visuele identiteit)

Wii Party-stijl behouden:

- Chunky witte outlines op 3D-objecten (BackSide trick, zie `makePlayerToken`)
- Cartoon toon shading (`MeshToonMaterial` met `gradientMap: toon`)
- Bouncy animaties met `cubic-bezier(.4,1.6,.5,1)` of `cubic-bezier(.3,1.7,.5,1)`
- Tekst-stempels met 4px witte border + dikke shadow
- Felle kleuren met flinke emissive intensity
- Sonore knoppen: goud-gradient `linear-gradient(180deg, #ffd54a, #ff9a1a)` met `box-shadow: 0 4px 0 #b86c00`
- Bewegingen overdrijven: `transform: scale(1.12) rotate(3deg)` bij POP-animaties
- Emoji als hoofd-iconografie — geen icon fonts

**Verboden:**

- Sleek/minimalistisch design
- Donkere mode-only UI
- Hairline borders
- Sans-serif fonts kleiner dan 11px

---


## 18. ⚡ NIEUW: Avatar systeem (mei 2026, v2)

[#18--nieuw-avatar-systeem-mei-2026-v2](#18--nieuw-avatar-systeem-mei-2026-v2)

Elke speler heeft een **chibi avatar** in plaats van een simpele kleurpion. Twee renderers naast elkaar: **SVG** voor alle 2D-contexten (Creator UI, HUD, scoreboards, Schnaps Call) en **echte 3D-meshes** voor de pion op het bord. Geen externe libraries, opslag in `localStorage`. CSS-prefix `av-`.

### v2 highlights

[#v2-highlights](#v2-highlights)

- **Real 3D heads op het bord** (geen sprite/CanvasTexture meer). Bol-hoofd in huidskleur met dark outline, eye/mouth/brow/nose als losse 3D-meshes, hair als 3D-geometrie per archetype, accessoires als 3D-primitives. Lost het "vierkante achtergrond-fringe" probleem op.
- **Pionnen kijken naar de camera (Y + X tilt)**. In `_gameHook` (onderaan `index.html`): de hele token roteert op de Y-as zodat de voorkant (gezicht) naar de camera draait, mét ±4° sway met per-player phase-offset (niet star, niet creepy-sync). DAARBOVENOP kantelt het *hoofd-group* op zijn X-as ~70% van de camera-elevatie hoek (minimaal 8°), zodat het gezicht zichtbaar is bij top-down camera. Plus ±2° nod. Vervangt de oude `time * 0.6` constante spin.
- **Soepelere camera-overgangen**. `autoRotateSpeed: 0.35 → 0.22`, `dampingFactor: 0.08 → 0.055`, idle-resume timer `4000 → 4500ms`. POV-volgcamera tijdens beurten: eerste-hop duur `420 → 540ms` met cubic-in-out i.p.v. cubic-out, vervolgstappen `320 → 360ms`, chase-lerps van `0.18/0.22` naar `0.13/0.17` (meer glide, minder snappy). Terug-naar-orbit ease `600 → 820ms` met cubic-in-out.
- **Dobbelsteen-bug opgelost**. In `dice-overlay.html`: oude spawn `x = ±[3.5, 5.0]` lag BUITEN de wall (radius 2.9), dus dobbelstenen vielen direct buiten de tafel. Vervangen door ring-spawn `r = 1.7…2.3` binnen de wall, throw richting tegenoverliggende punt met `±35°` spread en `3.6…6.0 m/s` (was 5-8). Angular velocity verlaagd `±22 → ±16 rad/s`. Plus een safety-clamp in `loop()` die elke dobbelsteen die toch buiten `r = 2.55` belandt terug naar binnen pakt en de uitwaartse velocity-component dempt.
- **Veel meer vrouwelijke opties**: glamour-ogen, lashed/cute_lashed/eyeliner, rode/roze lippen, soft_smile, lips_full, en 6 nieuwe feminiene haarstijlen (braids, bob, space_buns, side_swept, messy_bun, half_up). 4 nieuwe accessoires (bow, flower, pearls, bunny_ears).
- **Nieuwe categorie "Wangen"** (☺): none, soft_pink, pink, red blush, sproeten (freckles), gouden sterren.
- **6 nieuwe gezichten** (triangle, narrow, wide_jaw, soft, angular, baby) → totaal 14 face shapes.
- **Schema versie nog steeds 1**, met `blush` als optionele extra property. Bestaande v1-saves blijven werken (oude saves zonder `blush` → wordt automatisch `'none'`).

### Architectuur

[#architectuur-18](#architectuur-18)

Vier lagen, allemaal inline in `index.html`:

1. **Schema + SVG-renderer** (~660 regels). Bovenaan na `function clear()`. Definieert alle constanten (`AVATAR_FACE_OPTIONS`, etc), `defaultAvatar` / `randomAvatar`, en de SVG part-renderers per categorie + `renderAvatarSVG(avatar, mode, extraAttrs)`.
2. **3D head builder** (~580 regels, na de Creator-code). De kern is `build3DAvatarHead(avatar) → THREE.Group`. Helpers: `parseHexColor`, `avMat` (cached material factory), `add3DOutline`, `applyFaceShapeScale`, `addEyes3D`, `addBrows3D`, `addNose3D`, `addMouth3D`, `addBlush3D`, `addHair3D`, `addAccessory3D`.
3. **Creator UI** (~330 regels). `openAvatarCreator(initial, title) → Promise<avatar|null>`. Full-screen overlay, Pattern A. Categorieën als horizontale scroll-tabs, opties als swipeable carousel met SVG mini-previews, kleurpalet als swatch-rij.
4. **Integratie helpers**:
   - `makePlayerAvatarToken(colorHex, avatar)` — base disc + body cone + skin-colored neck cylinder + 3D head group.
   - `renderPlayerAvatarDOM(player, size, mode)` — DOM-helper voor minigame-overlays.

### Avatar schema

[#avatar-schema](#avatar-schema)

```
{
  v: 1,
  face:  'round'|'oval'|'square'|'heart'|'long'|'chubby'|'pointed'|'diamond'
        |'triangle'|'narrow'|'wide_jaw'|'soft'|'angular'|'baby',        // 14
  skin:  '#hex',
  eyes:  { shape: 'normal'|'big'|'sleepy'|'wink'|'star'|'dot'|'closed'|'square'
                 |'sparkle'|'tired'|'lashed'|'glamour'|'cute_lashed'|'eyeliner',  // 14
          color: '#hex' },
  brows: { shape: 'flat'|'arched'|'angry'|'surprised'|'thick'|'thin'|'wavy'|'none',
          color: '#hex' },
  nose:  'small'|'button'|'wide'|'long'|'pointy'|'none',
  mouth: 'smile'|'grin'|'neutral'|'open'|'smirk'|'tongue'|'o'|'frown'|'kiss'|'tooth'
        |'lips_red'|'lips_pink'|'soft_smile'|'lips_full',                          // 14
  hair:  { style: 'bald'|'short_messy'|'short_neat'|'side_part'|'long_straight'
                 |'ponytail'|'pigtails'|'bun'|'curly'|'mohawk'|'bowl'|'afro'
                 |'spiky'|'wavy_long'|'braids'|'bob'|'space_buns'|'side_swept'
                 |'messy_bun'|'half_up',                                            // 20
          color: '#hex' },
  accessory: 'none'|'glasses'|'shades'|'cap'|'beanie'|'crown'|'bandana'|'headband'
            |'helmet'|'earrings'|'bow'|'flower'|'pearls'|'bunny_ears',             // 14
  body:  'slim'|'normal'|'chunky',
  outfit: '#hex',
  blush: 'none'|'soft_pink'|'pink'|'red'|'freckles'|'stars',                       // 6
}
```

Totaal: 14×14×14×20×14×6 = ~14 miljoen feature-combinaties (los van kleuren), dus echt iedereen krijgt een unieke avatar.

### Render-modi (SVG)

[#render-modi-svg](#render-modi-svg)

`renderAvatarSVG(avatar, mode)`:

- `'full'` — head + chibi-torso (viewBox `0 0 200 260`). Voor Creator preview, modals, Schnaps-call beller-portret.
- `'head'` — alleen kop (viewBox `20 0 160 180`). Voor option-thumbs, scoreboards, HUD.
- `'square'` — kop in 1:1 viewBox (`0 -10 200 200`). (Legacy van v1, niet meer gebruikt op het 3D-bord nu we 3D-meshes hebben — wel bruikbaar als avatar-icon op vierkante badges.)

### 3D head — coördinaten en geometrie

[#3d-head--coordinaten-en-geometrie](#3d-head--coordinaten-en-geometrie)

`build3DAvatarHead(avatar) → THREE.Group` bouwt een hoofd dat per default gecentreerd is op `(0, 0, 0)` met `headR = 0.34`. De caller positioneert deze op `y = 1.50` boven de body cone in `makePlayerAvatarToken`.

Vaste posities (relatief tot head-center, units = world units):
| Feature | x | y | z |
| --- | --- | --- | --- |
| Eyes (centers) | `±0.32 × headR` | `+0.15 × headR` | `+0.93 × headR` |
| Brows (centers) | `±0.32 × headR` | `+0.42 × headR` | `+0.91 × headR` |
| Nose | `0` | `-0.05 × headR` | `+1.02 × headR` |
| Mouth | `0` | `-0.38 × headR` | `+0.92 × headR` |
| Blush dots | `±0.60 × headR` | `-0.18 × headR` | `+0.88 × headR` |

Face-shape variatie via `applyFaceShapeScale(head, faceShape)`: non-uniform scaling van de hoofd-sphere. Bv. oval: `0.93/1.08/1`, long: `0.88/1.16/0.96`, chubby: `1.14/0.95/1.08`. De features zelf staan op vaste posities — wijzigen niet mee met de scale (anders krijg je rare vertekeningen).

Skin/hair/accessory materialen worden gecached via `avMat(type, color)` (key = `type + ':' + color`). Met 8 spelers en ~5 unieke kleuren per avatar zijn dat ~40 materialen, prima.

### Hair archetypes — mapping van SVG-stijl naar 3D-mesh

[#hair-archetypes--mapping-van-svg-stijl-naar-3d-mesh](#hair-archetypes--mapping-van-svg-stijl-naar-3d-mesh)

20 SVG-styles worden in `addHair3D` op simpelere 3D-geometrie gemapt:

| 3D archetype | Geometrie | SVG-styles die het gebruiken |
| --- | --- | --- |
| Skullcap | half-sphere boven hoofd | short_messy, short_neat, bowl, half_up |
| Skullcap + side bump | half-sphere + extra strand | side_part |
| Skullcap + curtain | half-sphere + open cilinder onder | long_straight, side_swept |
| Skullcap + wavy curtain | + wave-spheres aan zijden | wavy_long |
| Skullcap + ponytail sphere | + uitgerekte sphere achter | ponytail |
| Skullcap + pigtails | + 2 uitgerekte spheres opzij | pigtails |
| Skullcap + bun | + sphere bovenop | bun |
| Skullcap + messy bun | + grote sphere + losse strands | messy_bun |
| Skullcap + 2 buns | + 2 spheres bovenop | space_buns |
| Cluster small spheres | 16 spheres in cirkel | curly |
| Big sphere | grote bol met outline | afro |
| Strip | box op midden-top | mohawk |
| Skullcap + spikes | + 7 conische spikes | spiky |
| Skullcap + 2 braids | + 2 cilinders met bumps | braids |
| Skullcap + medium curtain | korter dan long | bob |

Visueel onderscheidend genoeg om op normale board-camera-afstand verschillen te zien.

### Integratiepunten in het hoofdspel

[#integratiepunten-in-het-hoofdspel](#integratiepunten-in-het-hoofdspel)

| Plek | Wat | Hoe |
| --- | --- | --- |
| **Setup-scherm player-row** | Klikbare avatar-thumb met edit-pencil (✎) | `renderAvatarSVG(p.avatar, 'head')` in `.av-thumb` div. Click → `openAvatarCreator`. |
| **Setup standalone tab** | Avatar Creator als "minigame" | Entry in `MINIGAMES` (`id:'avatar'`). Handler bovenaan `launchStandaloneMinigame`. |
| **Bord (3D)** | 3D head op de pion | `makePlayerAvatarToken(colorHex, avatar)`. Head-group at `y = 1.50`, neck cylinder at `y = 1.20`, body cone at `y = 0.68`. |
| **HUD bottom-card** | Avatar binnen identity-color ring | `refreshHUD` rendert `renderAvatarSVG(cur.avatar, 'head')` in de `.player-card .avatar` div. Glow blijft `cur.color.css`. |
| **Schnaps Call** | Beller-portret = avatar i.p.v. 🥃 | `playSchnapsCall`: in-game mode rendert `renderAvatarSVG(player.avatar, 'full')` in `.sk-avatar`. |
| **Standalone minigames** | Fake spelers krijgen avatar | `launchStandaloneMinigame` zet `avatar: loadAvatarForColor(i)` op de fake players. |
| **Avatar Creator (standalone)** | Sla op in slot 0 | Speciale `id === 'avatar'` branch in `launchStandaloneMinigame`. |

### Persistence-gedrag

[#persistence-gedrag](#persistence-gedrag)

- **Per kleurslot, niet per speler-naam**. Slot 0 (Rood) heeft één opgeslagen avatar, slot 1 (Blauw) een andere, etc.
- `localStorage` key `drank-eiland-avatars-v1`. JSON-array `[avatar0, avatar1, ...]`.
- Wordt automatisch geladen op setup → players kunnen meteen herkend worden.
- Wordt opgeslagen bij elke `Opslaan`-klik in de Creator.
- Bestaande v1-saves blijven werken: missing fields (zoals `blush`) → fallback via `defaultAvatar(...)` en property-defaulting in `addBlush3D`.

### Performance-noten

[#performance-noten](#performance-noten)

- SVG-rendering is goedkoop (gewoon string-concat). Geen `requestAnimationFrame` nodig.
- 3D heads worden 1× gebouwd per pion bij `makePlayerAvatarToken` en blijven in de scene. Wijzigt de avatar tijdens een spel niet → geen rebuild nodig.
- Materialen gecached in `_avMatCache` (Map). Per avatar ~5 unieke kleuren × ~7 types = ~35 materials max.
- Hair "curly" gebruikt 16 small spheres → met 8 spelers max 128 spheres voor curly. Acceptabel op moderne iPhones; bij prestatie-issues kan dit naar 8 spheres.
- Geen `avatarToCanvas` / `getAvatarTexture` meer nodig voor het 3D-bord. Die functies zijn weg samen met `_avatarTextureCache`.

### Een nieuwe avatar-categorie toevoegen — stappen

[#een-nieuwe-avatar-categorie-toevoegen--stappen](#een-nieuwe-avatar-categorie-toevoegen--stappen)

1. Definieer nieuwe constanten bovenaan (bv. `AVATAR_PIERCING_OPTIONS = ['none','nose','lip','brow']`).
2. Voeg property toe in `defaultAvatar` en `randomAvatar`.
3. Schrijf een SVG part-renderer `renderPiercing(kind)` die een SVG-fragment teruggeeft.
4. Voeg de renderer-aanroep in `renderAvatarSVG()` toe in de juiste z-volgorde.
5. Voeg een entry toe aan `AVATAR_CATEGORIES`.
6. Voeg Dutch labels toe in `AVATAR_LABELS`.
7. **Voor 3D**: schrijf `addPiercing3D(parent, piercing, headR)` en roep hem aan in `build3DAvatarHead`.
8. (Optioneel) Migratie-default in `loadAvatars()` als bestaande saves het veld missen.

### Bekende beperkingen / TODO's

[#bekende-beperkingen--todos](#bekende-beperkingen--todos)

- **Iframes** (Raft Battle, Bowling, Connect 4) zien de avatars nog niet — ontvangen alleen `name + color` via `postMessage`. v3-werk: avatar-payload meesturen en in iframe-side renderen.
- Geen positie/grootte-sliders voor fijnafstemming (eyes spacing, mouth size). Bewust v2-scope.
- 3D-features (eyes, brows, etc.) staan op vaste posities op de bol — als je extreem niet-uniform face-shape-scaling toepast (bv. `1.5×0.6×1`) kunnen ze van het oppervlak loslaten. Huidige scales blijven binnen ±18% dus geen probleem.

### Visuele identiteit (style anchor)

[#visuele-identiteit-style-anchor](#visuele-identiteit-style-anchor)

Match met Wii Party / bestaande UI:
- Chunky outline `stroke="#1a1010" stroke-width="2.5..3"` op ALLE SVG-shapes.
- Witte highlights in ogen (`circle r="1.8" fill="#fff"`).
- Felle skin tones, geen gradients (toon-style flat colors).
- 3D: `MeshToonMaterial` met `gradientMap: toon`, `emissive: color, emissiveIntensity: 0.10-0.25` voor cell-shaded look.
- 3D outlines: BackSide-trick met `MeshBasicMaterial { color: 0x1a1010 }` op scale ~1.06-1.10 (matches SVG stroke).
- Body cone behoudt witte outline (color identity), head krijgt dark outline (Wii-Mii style contrast tegen huidskleur).
- Geen schaduwen of gradients in de SVG zelf — diepte komt uit de stroke + `filter:drop-shadow` op de container.

---


## 19. Ten slotte

Als deze handleiding inconsistent is met de code: **de code wint**. Update dit document als je iets fundamenteel wijzigt aan de architectuur.

Begin altijd met de huidige `index.html` lezen vóór wijzigingen — er kan ondertussen iets bijgekomen zijn dat hier niet beschreven staat.

🌋🍻 — Drank Eiland team
