# Assistant Tarot - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a mobile-first web app (single `index.html`) to score French Tarot games, supporting 3/4/5 players with full FFT-compliant scoring.

**Architecture:** Single HTML file with embedded CSS and JS. State managed in a JS object, persisted to localStorage. Two-tab UI (new hand / game overview). No framework, no build step.

**Tech Stack:** HTML5, CSS3 (flexbox/grid, CSS variables), vanilla JavaScript (ES6+)

**Design doc:** `docs/plans/2026-02-15-tarot-scorer-design.md`

---

### Task 1: Scoring Engine (JS logic only)

**Files:**
- Create: `index.html` (script section only, no UI yet)

**Step 1: Write the scoring engine with inline test assertions**

Write the core `calculateScore()` function and a `runTests()` function that validates it against known scenarios. This is the heart of the app — get the math right first.

```javascript
// ---- SCORING ENGINE ----

const SEUILS = { 0: 56, 1: 51, 2: 41, 3: 36 };
const MULTIPLICATEURS = { prise: 1, garde: 2, gardeSans: 4, gardeContre: 6 };

function calculateScore(donne) {
  const { contrat, bouts, pointsPreneur, petitAuBout, poignee, chelem } = donne;

  const seuil = SEUILS[bouts];
  const mult = MULTIPLICATEURS[contrat];
  const diff = pointsPreneur - seuil;
  const gagné = diff >= 0;

  // Score de base : (25 + écart) × multiplicateur, négatif si chuté
  let scoreBase = (25 + Math.abs(diff)) * mult;
  if (!gagné) scoreBase = -scoreBase;

  // Petit au bout : 10 × multiplicateur
  let primePetit = 0;
  if (petitAuBout === 'attaque') primePetit = 10 * mult;
  else if (petitAuBout === 'defense') primePetit = -10 * mult;

  // Poignée : valeur fixe, toujours pour le camp vainqueur
  const POIGNEE_VALS = { simple: 20, double: 30, triple: 40 };
  let primePoignee = 0;
  if (poignee && poignee.niveau !== 'non') {
    const val = POIGNEE_VALS[poignee.niveau];
    // La poignée va toujours au camp vainqueur, peu importe qui l'a déclarée
    primePoignee = gagné ? val : -val;
  }

  // Chelem
  let primeChelem = 0;
  if (chelem === 'annonceReussi') primeChelem = 400;
  else if (chelem === 'nonAnnonceReussi') primeChelem = 200;
  else if (chelem === 'annonceChuté') primeChelem = -200;

  const totalPreneur = scoreBase + primePetit + primePoignee + primeChelem;

  return totalPreneur;
}
```

**Step 2: Write test assertions**

```javascript
function runTests() {
  const tests = [
    // Garde, 2 bouts, 45 pts => gagné de 4 => (25+4)*2 = 58
    { input: { contrat: 'garde', bouts: 2, pointsPreneur: 45, petitAuBout: 'non', poignee: null, chelem: 'non' }, expected: 58 },
    // Prise, 0 bout, 50 pts => chuté de 6 => -(25+6)*1 = -31
    { input: { contrat: 'prise', bouts: 0, pointsPreneur: 50, petitAuBout: 'non', poignee: null, chelem: 'non' }, expected: -31 },
    // Garde sans, 3 bouts, 36 pts => juste fait => (25+0)*4 = 100
    { input: { contrat: 'gardeSans', bouts: 3, pointsPreneur: 36, petitAuBout: 'non', poignee: null, chelem: 'non' }, expected: 100 },
    // Garde, 1 bout, 56 pts, petit au bout attaque => (25+5)*2 + 10*2 = 80
    { input: { contrat: 'garde', bouts: 1, pointsPreneur: 56, petitAuBout: 'attaque', poignee: null, chelem: 'non' }, expected: 80 },
    // Prise, 2 bouts, 41 pts, poignée simple (attaque), juste fait => 25*1 + 20 = 45
    { input: { contrat: 'prise', bouts: 2, pointsPreneur: 41, petitAuBout: 'non', poignee: { niveau: 'simple', camp: 'attaque' }, chelem: 'non' }, expected: 45 },
    // Garde contre, 1 bout, 40 pts => chuté de 11 => -(25+11)*6 = -216, poignée défense => -216 - 20 = -236
    { input: { contrat: 'gardeContre', bouts: 1, pointsPreneur: 40, petitAuBout: 'non', poignee: { niveau: 'simple', camp: 'defense' }, chelem: 'non' }, expected: -236 },
    // Chelem annoncé réussi, garde, 3 bouts, 91 pts => (25+55)*2 + 400 = 560
    { input: { contrat: 'garde', bouts: 3, pointsPreneur: 91, petitAuBout: 'non', poignee: null, chelem: 'annonceReussi' }, expected: 560 },
  ];

  let passed = 0;
  tests.forEach((t, i) => {
    const result = calculateScore(t.input);
    if (result === t.expected) {
      console.log(`✓ Test ${i + 1} passed`);
      passed++;
    } else {
      console.error(`✗ Test ${i + 1} FAILED: expected ${t.expected}, got ${result}`);
    }
  });
  console.log(`${passed}/${tests.length} tests passed`);
}

runTests();
```

**Step 3: Open in browser, check console**

Run: `open index.html` then open browser console (Cmd+Option+J)
Expected: `7/7 tests passed`

**Step 4: Add score distribution function**

```javascript
function distributeScores(totalPreneur, nbJoueurs, preneurIndex, partenaireIndex) {
  const scores = new Array(nbJoueurs).fill(0);

  if (nbJoueurs === 3) {
    scores[preneurIndex] = totalPreneur * 2;
    for (let i = 0; i < 3; i++) {
      if (i !== preneurIndex) scores[i] = -totalPreneur;
    }
  } else if (nbJoueurs === 4) {
    scores[preneurIndex] = totalPreneur * 3;
    for (let i = 0; i < 4; i++) {
      if (i !== preneurIndex) scores[i] = -totalPreneur;
    }
  } else if (nbJoueurs === 5) {
    scores[preneurIndex] = totalPreneur * 4;
    scores[partenaireIndex] = totalPreneur;
    for (let i = 0; i < 5; i++) {
      if (i !== preneurIndex && i !== partenaireIndex) scores[i] = -totalPreneur;
    }
  }

  return scores;
}
```

**Step 5: Add distribution tests**

```javascript
// In runTests(), add:
// 4 joueurs, preneur index 0, total 58 => [174, -58, -58, -58]
const dist4 = distributeScores(58, 4, 0, null);
console.assert(dist4[0] === 174 && dist4[1] === -58, 'Distribution 4j failed');

// 5 joueurs, preneur index 0, partenaire index 2, total 80 => [320, -80, 80, -80, -80]
const dist5 = distributeScores(80, 5, 0, 2);
console.assert(dist5[0] === 320 && dist5[2] === 80 && dist5[1] === -80, 'Distribution 5j failed');

// 3 joueurs, preneur index 1, total -31 => [31, -62, 31]
const dist3 = distributeScores(-31, 3, 1, null);
console.assert(dist3[1] === -62 && dist3[0] === 31, 'Distribution 3j failed');

// Vérifier somme = 0
console.assert(dist4.reduce((a, b) => a + b, 0) === 0, 'Sum 4j != 0');
console.assert(dist5.reduce((a, b) => a + b, 0) === 0, 'Sum 5j != 0');
console.assert(dist3.reduce((a, b) => a + b, 0) === 0, 'Sum 3j != 0');
```

**Step 6: Verify all tests pass in console, then commit**

Run: `open index.html`, check console
Then: `git add index.html && git commit -m "feat: add scoring engine with test assertions"`

---

### Task 2: Game State Management

**Files:**
- Modify: `index.html` (add state management to script section)

**Step 1: Add game state object and localStorage persistence**

```javascript
// ---- STATE MANAGEMENT ----

const DEFAULT_STATE = {
  nbJoueurs: 4,
  joueurs: ['Joueur 1', 'Joueur 2', 'Joueur 3', 'Joueur 4'],
  donnes: [],       // Array of { donne params + scores[] }
  partieActive: false
};

let state = loadState();

function loadState() {
  try {
    const saved = localStorage.getItem('tarot-state');
    return saved ? JSON.parse(saved) : { ...DEFAULT_STATE };
  } catch {
    return { ...DEFAULT_STATE };
  }
}

function saveState() {
  localStorage.setItem('tarot-state', JSON.stringify(state));
}

function nouvellePartie(nbJoueurs, noms) {
  state = {
    nbJoueurs,
    joueurs: noms.slice(0, nbJoueurs),
    donnes: [],
    partieActive: true
  };
  saveState();
}

function ajouterDonne(donneParams) {
  const total = calculateScore(donneParams);
  const scores = distributeScores(total, state.nbJoueurs, donneParams.preneurIndex, donneParams.partenaireIndex);
  state.donnes.push({ ...donneParams, total, scores });
  saveState();
  return { total, scores };
}

function annulerDerniereDonne() {
  if (state.donnes.length > 0) {
    state.donnes.pop();
    saveState();
    return true;
  }
  return false;
}

function getScoresCumules() {
  const cumuls = new Array(state.nbJoueurs).fill(0);
  state.donnes.forEach(d => {
    d.scores.forEach((s, i) => { cumuls[i] += s; });
  });
  return cumuls;
}
```

**Step 2: Test state management in console, commit**

Verify in console: `nouvellePartie(4, ['Alice','Bob','Chloé','David'])`, then `ajouterDonne(...)`, then `getScoresCumules()`, then refresh page and verify state persists.

Then: `git add index.html && git commit -m "feat: add game state management with localStorage"`

---

### Task 3: HTML Structure & CSS Styling

**Files:**
- Modify: `index.html` (add HTML body and CSS)

**Step 1: Write the full HTML structure**

The page has:
- A header with app title
- A setup screen (shown when no active game)
- Tab navigation (Nouvelle donne / Partie en cours)
- The "Nouvelle donne" form
- The "Partie en cours" view with score table, classement, and controls
- A modal for hand detail view

**Step 2: Write mobile-first CSS**

Key design choices:
- CSS variables for the color palette (dark green `#1a5c2a`, red `#c0392b`, cream `#f5f0e8`, dark `#2c3e50`)
- `max-width: 480px; margin: auto` for centered mobile layout
- Large touch targets (min 44px height)
- Card-style sections with subtle shadows
- Sticky header and tab bar
- Score table with horizontal scroll if needed
- Positive scores in green, negative in red
- Sans-serif font stack

**Step 3: Verify layout in browser, commit**

Run: `open index.html`, check layout on mobile viewport (Chrome DevTools toggle device toolbar)
Then: `git add index.html && git commit -m "feat: add HTML structure and mobile-first CSS"`

---

### Task 4: Setup Screen & Tab Navigation (JS interactivity)

**Files:**
- Modify: `index.html` (add UI logic)

**Step 1: Implement setup screen logic**

- Number of players selector (3/4/5 buttons)
- Dynamic player name inputs based on selection
- "Commencer la partie" button → calls `nouvellePartie()`, hides setup, shows tabs

**Step 2: Implement tab switching**

- Click on tab → show corresponding panel, hide the other
- Active tab highlighted with CSS class

**Step 3: Implement "Nouvelle partie" button**

- In the "Partie en cours" tab, the "Nouvelle partie" button shows a confirmation dialog
- On confirm → reset state, show setup screen again

**Step 4: Test flow in browser, commit**

Run: `open index.html`, test creating a game, switching tabs, starting a new game.
Then: `git add index.html && git commit -m "feat: add setup screen and tab navigation"`

---

### Task 5: "Nouvelle donne" Form

**Files:**
- Modify: `index.html`

**Step 1: Wire up the form fields**

- Preneur: `<select>` populated with player names
- Partenaire: `<select>` shown/hidden based on `state.nbJoueurs === 5`, filtered to exclude preneur
- Contrat: 4 radio-style buttons (Prise/Garde/Garde Sans/Garde Contre)
- Bouts: 4 radio-style buttons (0/1/2/3)
- Points: `<input type="number" min="0" max="91">` with +/- stepper buttons
- Petit au bout: 3 radio-style buttons (Non/Attaque/Défense)
- Poignée: dropdown or button group (Non/Simple/Double/Triple), with sub-choice attaque/défense
- Chelem: dropdown (Non/Annoncé réussi/Non annoncé réussi/Annoncé chuté)

**Step 2: Add live score preview**

As the user fills the form, compute and display the score in real time at the bottom:
- "Score preneur : +58" (in green or red)
- Show the breakdown: "Base: 58 | Petit au bout: +20 | Poignée: +20"

**Step 3: Add form validation and submission**

- Validate: preneur selected, contrat selected, bouts selected, points entered (0-91)
- On submit: call `ajouterDonne()`, show success feedback, switch to "Partie en cours" tab, reset form

**Step 4: Test the full form flow, commit**

Run: `open index.html`, fill the form, verify preview, submit, check console for correct scores.
Then: `git add index.html && git commit -m "feat: add nouvelle donne form with live preview"`

---

### Task 6: "Partie en cours" View

**Files:**
- Modify: `index.html`

**Step 1: Implement score table rendering**

- Render function called after each donne and on page load
- Table: header row with player names, one row per donne (showing per-player score for that donne), footer row with cumulative totals in bold
- Positive scores in green, negative in red
- Highlight the preneur's cell in each row

**Step 2: Implement classement**

- Below the table, show players sorted by cumulative score descending
- Simple list with rank, name, score

**Step 3: Implement hand detail on row tap**

- Tapping a row shows a small modal/expandable section with: preneur, contrat, bouts, points, primes, and per-player scores for that hand

**Step 4: Implement "Annuler dernière donne"**

- Button calls `annulerDerniereDonne()`, re-renders the table
- Disabled if no donnes

**Step 5: Test the full game flow, commit**

Run: `open index.html`, play through several hands, check table, classement, detail view, undo.
Then: `git add index.html && git commit -m "feat: add partie en cours view with table, classement, and undo"`

---

### Task 7: Final Polish & Cleanup

**Files:**
- Modify: `index.html`

**Step 1: Remove test assertions from production code**

Remove `runTests()` call and test code (keep `calculateScore` and `distributeScores` clean).

**Step 2: Add page meta tags**

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta name="theme-color" content="#1a5c2a">
<meta name="apple-mobile-web-app-capable" content="yes">
<title>Assistant Tarot</title>
```

**Step 3: Final visual check on mobile**

Open in Chrome DevTools mobile view, test complete flow:
1. Create game with 4 players
2. Play 3 hands with different scenarios (win, loss, with primes)
3. Check table, classement, detail
4. Undo a hand
5. Switch to 5 players, verify partner field appears
6. Switch to 3 players, verify it works

**Step 4: Commit**

`git add index.html && git commit -m "feat: final polish, remove test code, add meta tags"`

---

## Summary

| Task | Description | Key Outcome |
|------|------------|-------------|
| 1 | Scoring engine | `calculateScore()` + `distributeScores()` with tests |
| 2 | State management | localStorage persistence, add/undo donnes |
| 3 | HTML + CSS | Full page structure, mobile-first styling |
| 4 | Setup + tabs | Player config, tab navigation |
| 5 | Donne form | Full form with live score preview |
| 6 | Game view | Score table, classement, detail, undo |
| 7 | Polish | Meta tags, cleanup, final testing |
