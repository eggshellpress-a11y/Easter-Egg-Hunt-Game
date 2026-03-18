# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development

No build step or server required. Open `index.html` directly in any modern browser (`open index.html` on macOS). All changes take effect on the next page reload. There are no tests, linters, or dependencies to install.

## Project Overview

This is a single-file browser game (`index.html`) — no build step, no package manager, no external dependencies. Open the file directly in a browser to run it. All CSS, JavaScript, and HTML live inline in that one file.

## File Structure

`index.html` is divided into three inline sections in order:

1. **`<style>`** — All CSS. Organized by screen/component with section header comments.
2. **`<body>`** — Two top-level screens (`#homeScreen`, `#gameScreen`) plus overlay divs (`#pauseOverlay`, `#exitOverlay`, `#gameOverOverlay`, `#casinoOverlay`, `#bonusEggOverlay`). Screen visibility is toggled via the `.hidden` CSS class.
3. **`<script>`** — All JavaScript. See sections below.

## JavaScript Architecture

The script is organized into clearly labeled sections (`// ===...===` comments):

| Section | Lines | Purpose |
|---|---|---|
| STATE | ~1556 | `cfg` (pre-game settings) and `game` (live game state) objects, plus all constants |
| AUDIO ENGINE | ~1611 | Web Audio API — procedural SFX functions + BGM scheduler + custom music upload |
| CONFETTI | ~1880 | Canvas overlay particle system |
| POLKADOT BACKGROUND | ~1991 | Animated polkadot canvas drawn behind game content |
| CASINO CELEBRATION | ~2038 | `playCasinoCelebration()` — slot-reel animation overlay (not triggered by start button; retained for potential use) |
| HOME SCREEN SETUP | ~2153 | `buildDecoEggs()`, avatar pickers, mode/difficulty selection builders |
| GAME START | ~2275 | `startGame()` — resets state, builds the grid, transitions screens |
| GRID BUILD | ~2359 | `buildGameHeader()`, `buildEggGrid()` — DOM construction for game screen |
| TIMER | ~2497 | `startTimer()`, `onTimerExpired()` — setInterval countdown |
| SHUFFLE | ~2527 | `doShuffle()` — burrow-out → reorder → pop-in sequence |
| EGG CLICK & GAME LOGIC | ~2628 | `onEggClick()`, `revealCard()`, `openEgg()`, `checkMatch()`, `updateBaskets()` |
| AI | ~2923 | `doAITurn()`, `updateAIMemory()` — memory-based AI with difficulty scaling |
| PAUSE | ~2983 | `togglePause()` — freezes timer and dims BGM gain |
| EXIT / GAME OVER | ~3024 | `confirmExit()`, `goHome()`, `onGameComplete()`, `restartGame()` |
| UTILS | ~3148 | `escHtml()` and other small helpers |
| INIT | ~3155 | One-time startup — default selections, toggle sync, volume slider init |

## Key State Objects

```js
cfg = {
  mode,        // '1p' | '2p' | 'ai'
  diff,        // 'easy' | 'medium' | 'hard'
  aiDiff,      // 'easy' | 'medium' | 'hard'  (AI memory usage: 0% / 50% / 80%)
  p1, p2,      // { name, avatar, photo }
  musicOn, sfxOn, customMusic,
  musicVolume, sfxVolume
}

game = {
  cards[],         // [{id, emoji, colorIdx, matched, revealed, el, wrapEl}]
  current[],       // indices of currently revealed (unmatched) cards (max 2)
  currentPlayer,   // 0 = p1, 1 = p2/AI
  scores[],        // [p1Score, p2Score]
  items[],         // [p1MatchedEmojis[], p2MatchedEmojis[]] — for basket display
  timer,           // seconds remaining
  maxTime,         // starting seconds (set from difficulty constants)
  paused,          // true when game is paused
  locked,          // true during animations — blocks all clicks
  aiMemory,        // { emoji: [cardId, cardId] } — AI's seen-card map
  totalPairs,      // total pairs in this round
  matchedPairs,    // pairs found so far
  shuffling,       // true during the post-timer shuffle sequence
}
```

## Egg Interaction Flow

1. Click → `onEggClick(idx)` — guards on `game.locked`, `matched`, `revealed`
2. → `revealCard(idx)` — plays squish CSS animation, then calls `openEgg()`
3. → `openEgg(card)` — sets `.open` class (top half rises), fires `burstConfetti()`
4. After 2nd card revealed → `checkMatch()`:
   - **Match**: burst + fade eggs out, add to basket, same player continues
   - **No match**: 1.5 s delay, `closeCard()` on both, turn switches

## Audio System

All sound is procedural Web Audio API — no audio files required. `audioCtx` is lazily created on first user gesture. BGM uses a scheduled melody loop (`scheduleMelody` / `scheduleBass` inside `startBGM()`). Custom uploaded music replaces BGM via `decodeAudioData`. SFX functions: `sfxPop`, `sfxMatch`, `sfxNoMatch`, `sfxTick`, `sfxWhoosh`, `sfxFanfare`, `sfxConfettiBurst`. A separate `sfxMasterGain` node controls SFX volume independently of BGM.

## Visual / Animation Conventions

- Eggs are CSS-only 3D shapes: `border-radius: 50% 50% 50% 50% / 60% 60% 40% 40%` with a `radial-gradient` body and `::before` gloss highlight.
- Each egg's palette is set as inline CSS custom properties (`--el`, `--em`, `--ed`, `--ea`) sourced from the `EGG_COLORS` array.
- Dance animation runs on `.egg-wrap` via `eggDance` keyframes; `animation-delay` is unique per egg so they never sync.
- Shuffle sequence is entirely CSS animation driven: `.burrow` → DOM reorder → `.popin` keyframes, gated by `animationend` events.
- The home screen decorative eggs are built by `buildDecoEggs()` using the first 5 entries of `EGG_COLORS`.

## Difficulty Constants

Defined in `DIFF_CONFIG`. Odd egg counts produce one unpaired `__LUCKY__` bonus egg.

| Level | Eggs | Pairs | Bonus egg | Timer |
|---|---|---|---|---|
| easy   | 15 | 7  | yes | 40 s |
| medium | 20 | 10 | no  | 30 s |
| hard   | 25 | 12 | yes | 20 s |

## Bonus Egg

When a `__LUCKY__` egg is clicked, `handleBonusEgg()` fires: gameplay locks, the `#bonusEggOverlay` fades in with a large CSS golden egg and rotating conic shimmer, confetti bursts at screen center, then the overlay auto-dismisses after 1.8 s and play resumes. No flash text is shown.

## UI Layout Notes

- Game header (`.game-header`) uses a three-column flex layout: `.header-left` (home + pause), `.header-center` (timer), `.header-right` (audio controls). Left and right each have `flex: 1` so the timer stays truly centered. The header has no background — buttons use `#c1f0fb` fill.
- The `.turn-indicator` (player turn display) sits between the baskets row and the egg grid, styled with a `#ff8eea` background and white border.
- Volume/SFX sliders on the home screen are wrapped in `.volume-row` divs with a solid white background. In-game audio controls live in `.header-right`.
- There is no flash message system — it was removed. Do not re-add `setFlash`/`clearFlash`.
