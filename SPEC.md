# TinyFingers Deluxe — Design Specification

A baby/toddler keyboard-smashing game as a **single self-contained static `index.html`** (no server, no external assets, no network requests, no analytics). Works by opening the file directly or from any static host.

## Product goals
1. **Delight**: every input (key, mouse, touch) produces an immediate, colorful, musical response.
2. **True lock mode** (the differentiator): once a parent starts play mode, a smashing baby cannot escape fullscreen, open menus, switch tabs, trigger browser/OS shortcuts, or navigate away. Only a parent can exit, via an on-screen code (no memorization needed).
3. **Zero infrastructure**: one HTML file, all CSS/JS inline, visuals from canvas + emoji + CSS, audio from WebAudio synthesis, optional voice from `speechSynthesis`. No external fonts, images, or scripts.

## Architecture (single file, three modules + shell)
Final `index.html` layout:
```
<style>       (all CSS, includes design tokens as CSS custom properties)
<markup>      (start screen, stage, canvas, overlays, parent UI)
<script>      one script tag, modules in this order:
   TF.audio   – sound + speech engine
   TF.engine  – visual engine (canvas particles + DOM glyphs + themes)
   shell      – lock system, input capture, state machine, parent gate, settings, wiring
window.TF = { audio, engine, settings, state }
```
Modules communicate ONLY through the interfaces below so they can be built in parallel.

### Shared settings object (owned by shell)
```js
TF.settings = {
  sound: true, speech: true, reducedMotion: false,
  theme: 'space',            // 'space' | 'ocean' | 'garden' | 'candy'
}
```
Persisted to `localStorage['tf-settings']` (wrapped in try/catch — must work with storage blocked, e.g. file:// or private mode). Shell calls `TF.engine.applySettings()` / `TF.audio.applySettings()` after changes.

### TF.audio public API (WebAudio + speechSynthesis, no assets)
- `TF.audio.init()` — create/resume AudioContext lazily **on first user gesture** (autoplay policy). Must be safe to call repeatedly.
- `TF.audio.onKey(info)` — `info = {key, code, isLetter, isDigit, comboLevel}`. Plays a short synthesized note. Letters map deterministically to a **pentatonic scale** across ~2 octaves (so smashing always sounds musical, and the same key always makes the same note — toddlers learn cause/effect). Digits → marimba-ish plucks; space → soft kick/thump; other keys → sparkle blips. Use short envelopes (attack ~5ms, decay ~300ms), a master gain ~0.5, and a limiter (DynamicsCompressor) so 10 simultaneous keys don't clip.
- `TF.audio.onPointer(x01, y01)` — gentle chime, pitch mapped to y.
- `TF.audio.speak(text)` — speechSynthesis at rate ~0.9, slightly high pitch; cancel pending utterances first so smashing doesn't queue 50 words. Respect `settings.speech`. Wrap in try/catch (API may be absent).
- `TF.audio.celebrate()` — short rising arpeggio for combo milestones.
- All methods must no-op cleanly when `settings.sound` is false or AudioContext is unavailable.

### TF.engine public API (visuals)
- `TF.engine.init(canvasEl, stageEl)` — sets up a full-viewport canvas (device-pixel-ratio aware, resize-safe) plus a DOM layer for big glyphs. Runs one `requestAnimationFrame` loop; pause the loop when idle (no particles, no glyphs) to save battery.
- `TF.engine.onKey(info)` —
  - **Letters**: giant glyph (~30–40vmin, bold, rounded system font stack) pops in at a random-ish position (kept fully on-screen) with a bounce-pop animation and bright color from the theme palette, plus a canvas particle burst behind it. Shell may follow with `TF.audio.speak('A')`.
  - **Digits**: glyph shows the digit AND that many small emoji items (e.g. 3 → "🐟🐟🐟") so it's mildly educational.
  - **Space / Enter**: full-screen confetti burst or big ripple.
  - **Other keys**: themed emoji (from the current theme's emoji set) pops with particles.
  - Glyphs live ~1.5s then animate out; cap concurrent DOM glyphs at ~12 (recycle oldest) so smashing can't grow the DOM unboundedly.
- `TF.engine.onPointerMove(x, y)` — sparkle trail particles.
- `TF.engine.onPointerDown(x, y)` — firework burst at the point.
- `TF.engine.combo(level)` — at combo milestones (see shell), screen-wide burst + brief background flash.
- `TF.engine.setTheme(name)` / `TF.engine.applySettings()` — 4 themes, each = background gradient (CSS custom properties on `<body data-theme=...>`), a 6–8 color palette, an emoji set, and a few slow ambient drifting background emoji (stars/fish/petals/candy). `reducedMotion: true` → no ambient drift, shorter/simpler animations, fewer particles.
- Performance: particle pool with hard cap (~400 particles), O(n) update, no per-frame allocations in the hot loop, no shadows/blur on canvas.

### Shell (lock system, input, state machine, parent gate)

**State machine**: `idle → playing(locked) ⇄ relock-pending → unlocked → (resume | exited)`

1. **Start screen (`idle`)**: friendly explanation for the parent, browser-capability note, big "Start playing 🔒" button. On click:
   - `document.documentElement.requestFullscreen({ navigationUI: 'hide' })`
   - `await navigator.keyboard.lock()` — **no arguments = capture ALL keys** (Esc, MetaLeft/Right, Tab, F1–F12, Ctrl/Alt combos…). Feature-detect; on non-Chromium continue without it and set `state.lockAPI = false`.
   - Generate session **parent code**: 4 random digits (allow repeats), regenerated each session.
   - Enter `playing`.
   - The start screen must state honestly: full lock needs Chrome/Edge/Chromium; on other browsers Esc exits fullscreen but auto-relock still catches it; and nothing on the web can block Ctrl+Alt+Del or (on some platforms) Alt+F4 — for absolute lockdown pair with OS kiosk mode.
2. **Playing**: every `keydown` gets `preventDefault()` + `stopPropagation()` (capture phase on `window`). Ignore key **repeats** for visuals/audio (`e.repeat`), but still preventDefault them. Route to engine/audio. Track stats: total presses, unique keys, start time, per-key counts, max combo. **Combo**: presses within a rolling 5s window; milestones at 25/50/100/200… trigger `engine.combo` + `audio.celebrate`.
3. **Escape-hatch counter (auto-relock)**: Chrome's unavoidable escape is *hold Esc 2s* (exits fullscreen). Listen to `fullscreenchange`: if fullscreen is lost while state is `playing`, switch to `relock-pending`: show an opaque full-viewport overlay ("🔒 Keep smashing to play!") that visually covers everything, and on the **next `keydown` or `pointerdown`** (each is a user activation) immediately re-`requestFullscreen()` + re-`lock()` and return to `playing`. Babies keep pressing keys, so escape lasts under a second. Also re-check/re-lock on `visibilitychange` → visible (needs the next user activation too — same overlay path). If `requestFullscreen` rejects (activation consumed), keep the overlay and retry on the next input.
4. **Belt-and-braces blocking while locked**: `contextmenu`, `selectstart`, `dragstart`, `wheel` with ctrlKey (zoom) → preventDefault (wheel listener must be `{passive:false}`); `beforeunload` returns a prompt while locked (safety net for Ctrl+W on non-Chromium); `touch-action: none` and `user-select: none` on the stage; `auxclick` prevented. History-trap: on entering `playing`, `history.pushState` a dummy entry and on `popstate` push again (defeats Backspace/mouse-back on non-Chromium).
5. **Parent gate (the only exit)**:
   - A small badge, bottom-right, ~35% opacity: "👨‍👩‍👧 Parents: type **4 8 2 9** to unlock" showing the live session code. It enlarges to full opacity when the mouse hovers/moves near it (babies smash keys; parents use the mouse). Digits shown with spaces so it reads as a code, not a word.
   - **Code entry**: strictly consecutive. Keep an expected-index pointer; a keydown of the next correct digit advances it, ANY other key resets it (to 1 if that key happens to be the first digit, else 0). Four consecutive correct digits → `unlocked`. No Enter needed. This makes accidental unlock by random smashing statistically negligible while staying trivial for a parent reading the badge.
   - **Mouse-only alternative**: press-and-hold the badge with the pointer for 2s → opens the unlock overlay with an on-screen keypad; clicking the 4 code digits also unlocks (same reset rule). Hold must survive `pointerleave` cancel; show a small radial progress while holding.
6. **Unlocked overlay**: dims the stage; shows the **Smash Report** (total keys smashed, minutes played, top 3 favorite keys, max combo, a fun "chaos level" title) plus buttons: **Resume play 🔒** (re-fullscreen + re-lock via that click's activation), **Exit** (calls `navigator.keyboard.unlock()`, `document.exitFullscreen()`, back to start screen), and settings toggles (sound, voice, theme picker, reduced motion). While unlocked, keys are NOT swallowed except the overlay ignores them; if no interaction for 30s, auto-resume lock is NOT possible without activation — instead just leave the overlay up (it still covers the game; a baby pressing keys does nothing because state isn't `playing`... but keys must still be preventDefault'd while the overlay is up and we're still in fullscreen+keyboard-lock, so browser shortcuts stay dead).
   - Important: do **not** release keyboard lock or fullscreen upon entering `unlocked` — only on explicit Exit. That way the machine stays safe while the parent reads the report.
7. **Idle-audio guard**: create/resume AudioContext inside the first trusted gesture handler; iOS/Safari need `resume()` after visibility changes.

### Markup inventory
`#start` (start screen) · `#stage` (play area: `#fx-canvas`, `#glyph-layer`, `#ambient-layer`, `#hud` [keys-smashed counter + combo meter, big rounded font, top-center, pointer-events:none]) · `#relock` (opaque relock overlay) · `#parent-badge` · `#unlock` (report + keypad + settings). All overlays toggled via a `data-state` attribute on `<body>` — CSS drives visibility from state, JS only sets the attribute.

### Visual design language
Playful, soft, high-contrast: rounded everything, big shapes, saturated palettes per theme on deep/dark gradients so colors pop. System font stack with `font-weight: 900` and generous letter-spacing for glyphs. Subtle CSS `@keyframes` for pop-in (scale 0.3→1.1→1 with overshoot). Respect `prefers-reduced-motion` by defaulting `reducedMotion` to true when the media query matches.

### Hard requirements / gotchas checklist
- `keydown` listener on `window` with `{capture:true}`; preventDefault EVERYTHING while in `playing`/`relock-pending`/`unlocked` states.
- `navigator.keyboard.lock()` MUST be called while fullscreen is active (or pending) — call it right after `requestFullscreen()` resolves, and again on every re-entry.
- Feature-detect everything: `navigator.keyboard?.lock`, `speechSynthesis`, `AudioContext || webkitAudioContext`, `localStorage`.
- No `console.log` left in, no TODOs, no external URLs anywhere in the file.
- Works from `file://` (no modules, no fetch, no service worker).
- Single script tag, strict mode, everything inside an IIFE except the `window.TF` namespace.
- Canvas resize on `resize`/`fullscreenchange` with DPR handling.
- The whole file must be valid standalone HTML5; target modern evergreen browsers only.
