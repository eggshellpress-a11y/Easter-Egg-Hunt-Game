# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development

No build step or server required. Open `index.html` directly in any modern browser (`open index.html` on macOS). All changes take effect on the next page reload. There are no tests, linters, or dependencies to install.

## Project Overview

This is a single-file browser game (`index.html`) — no build step, no package manager, no external dependencies (except PeerJS loaded from CDN for online mode). Open the file directly in a browser to run it. All CSS, JavaScript, and HTML live inline in that one file (~5060 lines).

## File Structure

`index.html` is divided into three inline sections in order:

1. **`<style>`** — All CSS. Organized by screen/component with section header comments.
2. **`<body>`** — Three top-level screens: `#homeScreen`, `#settingsScreen`, `#gameScreen`. Overlay divs: `#pauseOverlay`, `#exitOverlay`, `#gameOverOverlay`, `#casinoOverlay`, `#bonusEggOverlay`, `#smsOverlay`, `#readyLobbyOverlay`, `#countdownOverlay`, `#onlineGameOverOverlay`, `#onlineDisconnectOverlay`. Chat bar: `#chatBar` (online only, default collapsed). Two global elements outside screens: `#screenBlurOverlay` (backdrop) and `#gameMessage` (floating match/no-match toast). Screen visibility is toggled via the `.hidden` CSS class.
3. **`<script>`** — All JavaScript. See sections below.

## JavaScript Architecture

The script is organized into clearly labeled sections (`// ===...===` comments):

| Section | Line | Purpose |
|---|---|---|
| STATE | ~2373 | `cfg` (pre-game settings) and `game` (live game state) objects, plus all constants |
| AUDIO ENGINE | ~2431 | Web Audio API — procedural SFX functions + BGM scheduler + custom music upload |
| CONFETTI | ~2700 | Canvas overlay particle system |
| POLKADOT BACKGROUND | ~2811 | Animated polkadot canvas drawn behind game content |
| CASINO CELEBRATION | ~2858 | `playCasinoCelebration()` — slot-reel animation overlay (retained for potential use) |
| HOME SCREEN SETUP | ~2973 | `buildDecoEggs()`, avatar pickers, mode/difficulty selection builders |
| GAME START | ~3101 | `startGame()` — resets state, builds the grid, transitions screens |
| GRID BUILD | ~3190 | `buildGameHeader()`, `buildEggGrid()` — DOM construction for game screen |
| SLIDE IN ANIMATION | ~3300 | `animateEggsIn()` — CSS-driven screen transition; uses `animationend` + `setTimeout` fallback |
| TIMER | ~3351 | `startTimer()`, `onTimerExpired()` — setInterval countdown (offline only) |
| SHUFFLE | ~3410 | `doShuffle()` — burrow-out → reorder → pop-in sequence; `startShuffleTimer()` / `onShuffleComplete()` |
| EGG CLICK & GAME LOGIC | ~3519 | `onEggClick()`, `revealCard()`, `openEgg()`, `checkMatch()`, `updateBaskets()` |
| AI | ~3841 | `doAITurn()`, `updateAIMemory()` — memory-based AI with difficulty scaling |
| PAUSE | ~3901 | `togglePause()` — freezes timer and dims BGM gain |
| EXIT / GAME OVER | ~3941 | `confirmExit()`, `onGameComplete()`, `restartGame()` |
| ONLINE MULTIPLAYER | ~3960 | PeerJS peer-to-peer — host/join lobby, move sync, chat, disconnect handling |
| ONLINE READY LOBBY | ~3973 | `showReadyLobby()`, ready countdown, `_startBothReady()` |
| ONLINE GAME OVER & REMATCH | ~4048 | `showOnlineGameOver()`, `offerRematch()`, `declineRematch()`, `startRematch()` |
| HIGH SCORE | ~4586 | `localStorage`-backed per-mode/diff high scores |
| SETTINGS SCREEN | ~4601 | `showSettings()`, `closeSettings()`, `goHome()` |
| UTILS | ~4772 | `launchMatchFireworks()`, `fireFirework()`, `showGameMessage()`, `escHtml()` |
| INIT | ~4962 | One-time startup — default selections, toggle sync, volume slider init, `?join=` URL param handling |

## Key State Objects

```js
cfg = {
  mode,        // '1p' | '2p' | 'ai' | 'online'
  diff,        // 'easy' | 'medium' | 'hard'
  aiDiff,      // 'easy' | 'medium' | 'hard'  (AI memory usage: 0% / 50% / 80%)
  p1, p2,      // { name, avatar, photo }
  musicOn, sfxOn, customMusic,
  musicVolume, sfxVolume
}

game = {
  cards[],              // [{id, emoji, colorIdx, matched, revealed, el, wrapEl}]
  current[],            // indices of currently revealed (unmatched) cards (max 2)
  currentPlayer,        // 0 = p1, 1 = p2/AI
  scores[],             // [p1Score, p2Score]
  items[],              // [p1MatchedEmojis[], p2MatchedEmojis[]] — for basket display
  timer,                // seconds remaining (offline modes)
  maxTime,              // starting seconds (set from difficulty constants)
  paused,               // true when game is paused
  locked,               // true during animations — blocks all clicks
  aiMemory,             // { emoji: [cardId, cardId] } — AI's seen-card map
  totalPairs,           // total pairs in this round
  matchedPairs,         // pairs found so far
  shuffling,            // true during the post-timer shuffle sequence
  shuffleTimer,         // seconds remaining on shuffle countdown (online mode)
  shuffleMaxTime,       // always 20 s in online mode
  shuffleTimerInterval, // setInterval handle for shuffle timer
}

// Online multiplayer module-level globals (not in game object)
peer                    // PeerJS Peer instance (null when offline)
onlineConn              // PeerJS DataConnection to opponent
isOnlineHost            // true if this client created the room
_isOnlineGame           // true while an online game is active
_rematchOffered         // this player clicked Rematch
_rematchOpponentOffered // opponent sent rematch_offer
_myReady                // this player clicked Ready in lobby
_opponentReady          // opponent sent player_ready
_skipInGameCountdown    // true when ready-lobby countdown already played (skip in-game one)
_pendingCountdown       // queued countdown callback if msg arrives before eggs finish animating
_autoLeaveTimer         // setTimeout handle for auto-return-to-home after decline/disconnect
```

## Egg Interaction Flow

1. Click → `onEggClick(idx, fromRemote)` — guards on `game.locked`, `matched`, `revealed`; `fromRemote=true` skips turn ownership check for online moves
2. → `revealCard(idx)` — plays squish CSS animation, then calls `openEgg()`
3. → `openEgg(card)` — sets `.open` class (top half rises), fires `burstConfetti()`
4. After 2nd card revealed → `checkMatch()`:
   - **Match**: burst + fade eggs out, add to basket, same player continues
   - **No match**: 1.5 s delay, `closeCard()` on both, turn switches

## Online Multiplayer

Uses PeerJS (loaded from CDN) for browser-to-browser WebRTC. The host generates a room code (`EGG` + 7 random chars) and shares it via clipboard copy or SMS (`sms:` URI). The joiner pastes the code or follows a `?join=<code>` URL.

**Full online game flow:**
1. **Lobby** (`#readyLobbyOverlay`): both players click Ready → `player_ready` message → `_startBothReady()` plays 3-2-1-PLAY! countdown, sets `_skipInGameCountdown = true`, starts game
2. **Game start**: host calls `startOnlineGameAsHost()` → syncs card order to guest via `{type:'init', cards:[...]}` → guest calls `applyOnlineInit()`
3. **CRITICAL**: `startShuffleTimer()` is called **before** `animateEggsIn()` in both host and guest paths. Do not move it inside the animation callback — on iOS Safari/mobile, `animationend` events and `setTimeout` fallbacks can be throttled or dropped, making any timer gated on the callback unreliable. The 20 s shuffle timer is long enough that eggs always finish animating (2.1 s) before the first shuffle fires.
4. **Shuffle loop**: `startShuffleTimer()` → 20 s countdown → `doShuffle()` (host only; guest receives `{type:'shuffle'}`) → `onShuffleComplete()` → `startShuffleTimer()` → repeat
5. **Moves**: each click broadcast as `{type:'click', idx}` so both peers see identical state; `fromRemote=true` skips turn ownership check
6. **Game over**: host detects `game.matchedPairs === game.totalPairs` → `sendOnline({type:'state', ...})` → both call `showOnlineGameOver()`
7. **Rematch**: `offerRematch()` sends `{type:'rematch_offer'}`; when both offered → `startRematch()` re-runs full flow; either declining sends `{type:'rematch_decline'}` → `showDeclineOverlay(msg)` → both auto-return home after 2.8 s

**PeerJS message types**: `init`, `countdown`, `ready_countdown`, `player_ready`, `click`, `shuffle`, `state`, `chat`, `leaving`, `rematch_offer`, `rematch_decline`

**Chat bar** (`#chatBar`): online only, starts collapsed. Toggle via `#chatToggle`. Easter color palette: pink→lavender→teal gradient, yellow (`#ffd93d`) accents.

**SMS sharing**: `#smsOverlay` / `#smsWidget` provides a consent gate before opening `sms:` URI; consent stored in `localStorage` as `smsConsentGiven`.

## Audio System

All sound is procedural Web Audio API — no audio files required. `audioCtx` is lazily created on first user gesture. BGM uses a scheduled melody loop (`scheduleMelody` / `scheduleBass` inside `startBGM()`). Custom uploaded music replaces BGM via `decodeAudioData`. SFX functions: `sfxPop`, `sfxMatch`, `sfxNoMatch`, `sfxTick`, `sfxWhoosh`, `sfxFanfare`, `sfxConfettiBurst`. A separate `sfxMasterGain` node controls SFX volume independently of BGM.

## Visual / Animation Conventions

- Eggs are CSS-only 3D shapes: `border-radius: 50% 50% 50% 50% / 60% 60% 40% 40%` with a `radial-gradient` body and `::before` gloss highlight.
- Each egg's palette is set as inline CSS custom properties (`--el`, `--em`, `--ed`, `--ea`) sourced from the `EGG_COLORS` array.
- Dance animation runs on `.egg-wrap` via `eggDance` keyframes; `animation-delay` is unique per egg so they never sync.
- Shuffle sequence is entirely CSS animation driven: `.burrow` → DOM reorder → `.popin` keyframes, gated by `animationend` events.
- The home screen decorative eggs are built by `buildDecoEggs()` using the first 5 entries of `EGG_COLORS`.
- Online game over overlay (`#onlineGameOverOverlay`) uses variant classes: `.ogo-winner` (gold shimmer, bouncing trophy), `.ogo-loser` (shake animation), `.ogo-tie` (pulse). Rematch UI lives in `.ogo-rematch-section`.
- Match/no-match feedback uses `showGameMessage(text, cls, duration)` — it populates `#gameMessage` and shows `#screenBlurOverlay`. There is no flash message system — do not re-add `setFlash`/`clearFlash`.
- Fireworks on match: `launchMatchFireworks()` spawns `.fw-particle` DOM elements at fixed screen positions and removes them after their CSS animation completes.

## Difficulty Constants

Defined in `DIFF_CONFIG`. Odd egg counts produce one unpaired `__LUCKY__` bonus egg.

| Level | Eggs | Pairs | Bonus egg | Timer |
|---|---|---|---|---|
| easy   | 15 | 7  | yes | 2:30 (150 s) |
| medium | 20 | 10 | no  | 3:30 (210 s) |
| hard   | 20 | 10 | no  | 4:30 (270 s) |

Online mode always uses a 20 s shuffle timer (not the per-difficulty timers). The in-game clock (`.clock-face`) is hidden in online mode; only the shuffle clock (`.shuffle-clock-face`) is shown.

## Bonus Egg

When a `__LUCKY__` egg is clicked, `handleBonusEgg()` fires: gameplay locks, the `#bonusEggOverlay` fades in with a large CSS golden egg and rotating conic shimmer, confetti bursts at screen center, then the overlay auto-dismisses after 1.8 s and play resumes.

## High Scores

Stored in `localStorage` keyed by `eggHunt_hs_{mode}_{diff}` (e.g. `eggHunt_hs_1p_easy`). `saveHighScore(score)` returns `true` if a new record was set. Scores are per mode+difficulty combination.

## UI Layout Notes

- **Home screen** (`#homeScreen`): mode selector (`#sectionMode`) which expands `#aiDiffRow` for AI mode or `#onlineLobby` for online mode; difficulty selector (`#sectionDiff`); settings gear button routes to `#settingsScreen`.
- **Settings screen** (`#settingsScreen`): third top-level screen (hidden by default) containing avatar/name pickers and all audio controls. `showSettings(from)` records the originating screen so `closeSettings()` can return to it.
- **Game header** (`.game-header`) has three columns: `.header-left` (home/exit egg button), `.header-center` (dual timer clocks), `.header-right` (`.mini-scores` — compact in-game score display, `#miniScores`). The exit button is a pink egg shape (`.home-egg-btn`).
- The header center contains a `.timers-row` with two clocks: `.timer-display` / `.clock-face` (game countdown, `#timerDisplay`, hidden in online mode) and `.shuffle-timer-display` / `.shuffle-clock-face` (shuffle countdown, `#shuffleTimerDisplay`, shown only in online mode).
- Audio controls (volume/SFX sliders, music upload) live on `#settingsScreen` only — there are no in-game audio controls.
