# WordFlow — Game Design Document

**Live game:** https://dailywordflow.com
**Repository:** https://github.com/Mscissors/wordflow (GitHub Pages, branch `main`, root)
**Entire game:** one file — `index.html` (embedded CSS + JS, no build step, no dependencies)

---

## 1. Game Overview

WordFlow is a daily browser word puzzle. Players are shown a vertical **chain of 8 words**. Only the **first word is revealed**; the remaining 7 are hidden, shown as blank tiles (one tile per letter). Every adjacent pair of words in the chain forms a **common two-word phrase** read top-to-bottom (e.g. SECRET → AGENT = "secret agent", AGENT → ORANGE = "agent orange"). The player solves the chain top-down, using each solved word as the clue for the next.

**Tagline / instructions (shown on the start screen):**
> CONNECT THE WORDS, BEAT THE CLOCK.
> - Each word makes a common phrase with the one above it
> - A new letter every 15 seconds, but each one is a strike.
> - Solve with the fewest strikes possible!

The game is deliberately **not framed as competitive** — it can be played solo against your own score. Avoid the word "wins" in copy. Sharing/comparison is opt-in.

## 2. Core Mechanics

### The chain
- 8 words per puzzle: 1 revealed anchor (top) + 7 hidden words.
- The player taps/clicks a hidden row to focus it, types a guess, presses Enter (or the Submit Guess button).
- Correct guess: the row flips green (staggered tile-flip animation) and focus auto-advances to the next unsolved word.
- Wrong guess: the row shakes and the typed letters clear. No penalty, no message.

### The timer & strikes
- A **15-second countdown** runs during play. The timer badge is docked directly to the **left of the currently focused row** and follows the focus. It turns red and pulses during the final 5 seconds.
- When the timer hits zero: the next sequential letter of the focused word is revealed (gold "hinted" tile, locked in), the **strike counter increments by 1**, and the timer restarts.
- The player can also press the **"💡 Next Letter" button** to reveal a letter voluntarily — same cost: 1 strike.
- Letter reveals preserve in-progress typing: the revealed letter consumes the tile where the player's *first* typed letter sat; remaining typed letters stay in place.
- If reveals complete a whole word, it auto-solves.
- Solving a word resets the timer to 15.
- The **strike counter** (red badge, ✖, top-right corner) is the score. Lower is better.

### Win state
- Puzzle is complete when all 7 hidden words are solved. Timer stops permanently.
- A victory modal shows the final strike count and a **Share** button.
- Share uses the native share sheet on mobile (Web Share API) and clipboard-copy on desktop. Format:
  `WordFlow #<puzzleNumber> | Strikes: <N> 🔴 | Can you beat me?` + the game URL.

### Start screen (intro overlay)
- Shown before play; the timer does NOT start until the player presses **▶ START**.
- Contains: app icon, WordFlow logo, "Puzzle #N · Month D, YYYY", instructions, a **mini demo board** (GAME solved green / SHOW solved green / HORSE half-revealed gold: H-O-R + 2 blanks, connected by ⛓ links), the **global average-strikes line**, and the pulsing START button.
- Players who already finished today skip the intro and see their victory modal.
- Players returning mid-game still get the intro (button always says "▶ START", never "Continue").

## 3. Daily Puzzle System

- **Global rollover at midnight US Eastern time** (`America/New_York` via `Intl.DateTimeFormat`, which handles EST/EDT automatically). Everyone on Earth flips to the new puzzle at the same instant. This is NOT per-timezone like Wordle.
- `LAUNCH_DATE = "2026-07-18"` (Eastern). Puzzle number = whole days since launch + 1.
- Puzzles live in the ordered `PUZZLES` array in `index.html`. The day's chain is `PUZZLES[offset % PUZZLES.length]` — the list **cycles** when exhausted, so the game never breaks. Extending the list extends the period before repeats.
- There are currently **21 chains** (see `PUZZLES` in `index.html` for the canonical list).

## 4. Persistence & Stats

### Local state (localStorage, key `wordflow-state`)
Saves per-day progress: solved words, revealed-letter counts, strike total, completed flag, reported flag. Keyed by Eastern date **and a chain signature** (`chain.join("-")`) — state is discarded if the date OR the chain changes, so editing a live puzzle safely resets players.

### Global stats (Firebase Realtime Database, REST — no SDK)
- Database: `https://wordflow-30aa2-default-rtdb.firebaseio.com` (project `wordflow-30aa2`, free Spark tier).
- Schema: `/daily/<YYYY-MM-DD> = { players: <count>, strikes: <total> }` — nothing else, fully anonymous.
- On win, the client PATCHes atomic increments: `{"players":{".sv":{"increment":1}},"strikes":{".sv":{"increment":N}}}`. The persisted `reported` flag ensures one report per player per day.
- The intro shows `Today's average: X.X strikes ✖` = `strikes / players` for today. If today has no players yet, it shows **yesterday's average + 2**. If neither day has data or the network fails, the line stays empty — stats are best-effort and never block gameplay.
- Security rules (published, no expiry): public read of `/daily`; `players` may only increment by exactly 1 per write; `strikes` may only increase, max +200 per write; no deletes/resets; everything else locked.

## 5. Visual Design

- Dark theme. CSS custom properties in `:root` — key colors: background `#0f1117`, accent blue `#5b8cff`, accent purple `#7c5bff`, solved green `#35c47c`, strike red `#ff4757`, hint gold `#ffd166`.
- Tile states: **anchor** (dark, blue border/text), **empty** (dark slate), **hinted** (gold, locked reveal), **typed** (player input, lighter border), **solved** (green). The intro demo board teaches these states visually.
- Mobile-first, max content width 480px. Typing on mobile works via a hidden input (`#ghost-input`) that summons the keyboard; desktop uses direct keydown.
- Animations: tile flip-in (`flipIn`, rotateX), row shake on wrong guess, strike-counter pop, timer pulse when low, START button breathing glow, modal fade/spring.
- Branding assets in repo: `wordflow-app-icon.png` (1024², tile-W with gold chain link), `wordflow-banner.png` (1200×630 Open Graph image), `favicon.png` (64), `apple-touch-icon.png` (180). Full social metadata is in the `<head>`.

## 6. Deployment & Operations

- Hosting: GitHub Pages from `main` branch root. **Deploy = `git push`** — live at dailywordflow.com within ~1 minute. No build step.
- Domain: `dailywordflow.com` via Porkbun; 4 A records → GitHub Pages IPs (185.199.108–111.153), CNAME `www` → `mscissors.github.io`, `CNAME` file in repo. HTTPS enforced, cert auto-renews.
- The old URL `mscissors.github.io/wordflow` redirects to the domain.

---

## 7. HOW TO AUTHOR NEW PUZZLES (guide for AI assistants)

New puzzles are appended to the `PUZZLES` array in `index.html`. Each entry is an array of 8 uppercase strings plus a comment listing the connecting phrases.

### Hard rules (every chain MUST satisfy all of these)
1. **Exactly 8 words.** Word 1 is the revealed anchor; words 2–8 are guessed.
2. **Every word is a standalone English dictionary word, 6 letters or fewer.** No proper nouns as words (KEY WEST is acceptable because KEY and WEST are both common words; the *phrase* may be a name, the *words* may not be obscure).
3. **Every adjacent pair, read top-to-bottom, forms a genuinely common two-word phrase or collocation.** "Common" means an average adult American would recognize it instantly: `secret agent`, `rush hour`, `bench press`, `game over`.
4. **NO glued compound words as the link.** The pair must work as a spoken/written two-word phrase. `RAIN → BOW` (rainbow) is forbidden; `SPACE → BAR` (space bar) is fine. Borderline cases that are commonly written both ways (`sting ray`) are acceptable but prefer unambiguous phrases.
5. **No word appears twice in the same chain.**
6. Family-friendly: no profanity, slurs, drugs, or innuendo-dependent phrases.

### Quality guidelines (strive for these)
- **Surprising turns make the fun.** The best chains pivot meaning mid-stream: `PARTY → FOUL → PLAY → DATE` or `KEY → WEST → COAST → GUARD`. A chain of same-topic words is boring.
- **Avoid ambiguous forks near the top of the chain.** If the anchor is DOG, the second word could be PARK, HOUSE, FOOD, BONE... The player only has letter-count + timer reveals to disambiguate; that's acceptable mid-chain but frustrating at word 2. Prefer anchors whose first link is strongly determined (SECRET → AGENT).
- **Mind the bottom of the chain**: there is no bottom anchor, so the final word is constrained only by the word above it. End on a strongly-bound phrase (`GAME → OVER`, `SOAP → OPERA`) rather than a loose one.
- Mix word lengths; 3-letter words (BOX, TAG, BAR) feel snappy between longer ones.
- Reusing a word across *different days* is allowed but avoid reusing a whole phrase pair, and vary anchors.
- Difficulty knob: shorter words + more famous phrases = easier. Aim for a solver finishing at roughly 3–8 strikes.

### Verification procedure (do this for every candidate chain before adding)
1. Read each adjacent pair aloud in order; confirm each is a real, common phrase *in that word order* (`PRESS BOX` and `BOX PRESS` are not interchangeable).
2. Check every word: ≤6 letters, real word, correctly spelled, uppercase.
3. Count: exactly 8 words, no duplicates.
4. Simulate a solve: for each hidden word, ask "knowing only the word above (and the letter count), is this guessable within a few 15-second reveals?" If a word is only obvious in retrospect, the chain is too hard.

### Adding puzzles to the game
1. In `index.html`, find `const PUZZLES = [`.
2. Append new entries at the **end** of the array (never reorder or edit *earlier* entries once their day has passed — the day-to-chain mapping is positional, and reordering changes which chain a given date serves).
3. Format (match existing style):
   ```js
   ["ANCHOR", "WORD2", "WORD3", "WORD4", "WORD5", "WORD6", "WORD7", "WORD8"],
   // anchor word2, word2 word3, word3 word4, ... (list all 7 phrases)
   ```
4. Editing **today's** chain mid-day is safe from a data-integrity standpoint (players' saved progress auto-resets via the chain signature) but resets everyone's progress — avoid unless necessary.
5. Deploy: `git add index.html && git commit -m "..." && git push`.

### Example of a compliant new chain
```js
["NIGHT", "LIFE", "GUARD", "RAIL", "ROAD", "TRIP", "WIRE", "TAP"],
// night life, life guard?  ← STOP: "lifeguard"/"guardrail"/"railroad" are glued
// compounds — this chain VIOLATES rule 4 and must be rejected.
```
```js
["ROYAL", "FAMILY", "TREE", "FROG", "LEG", "ROOM", "KEY", "WEST"],
// royal family, family tree, tree frog, frog leg(s), leg room,
// room key, key west  ← all spoken two-word phrases: ACCEPTED.
```

---

## 8. Known Constraints & Future Ideas

- Stats are unauthenticated and spoofable by a motivated actor (acceptable for current scale).
- Puzzle list is public in the page source — solutions are viewable by anyone who opens dev tools (inherent to serverless design; acceptable).
- The 21-day cycle repeats silently; extend `PUZZLES` before day 22 (2026-08-08) to avoid repeats.
- Ideas floated but not built: streak tracking, strike-count distribution graph ("you beat X% of players"), archive of past puzzles, hard mode (10-second timer), first-letter-free variant for the final word.
