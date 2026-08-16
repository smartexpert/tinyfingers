# KeySplash

A keyboard-smashing game for babies and toddlers — with a **real lock mode**. One self-contained `index.html`: no server, no build step, no external assets, no analytics, no network requests at all.

## Quick start

Open `index.html` in **Chrome or Edge** (any Chromium browser), click **Start playing 🔒**, and hand over the keyboard. That's it — it also works straight from a double-click on the file (`file://`).

## Why this exists

Existing keyboard-smash sites can't stop a baby from pressing the Windows key, F-keys, Ctrl+W, Alt+Tab, Esc… and suddenly the game is gone and the Start menu is open. This version uses the **Keyboard Lock API** (`navigator.keyboard.lock()`) in JS-initiated fullscreen, which captures essentially the whole keyboard — Esc, the Windows/Command key, Tab, F1–F12, Ctrl/Alt shortcuts — while playing.

### The lock model

- **Chrome's one built-in escape hatch** is holding Esc for ~2 seconds. The game counters it: the instant fullscreen is lost, an opaque "keep smashing to play" overlay appears, and the very next keypress or tap re-enters fullscreen and re-locks. A smashing baby re-seals the lock in under a second.
- **Parent exit, no memorization**: a random 4-digit code is generated each session and shown in the bottom-right badge (it brightens when you move the mouse near it — babies smash keys, parents use the mouse). Type the four digits consecutively; any wrong key resets progress, so random smashing essentially can't hit it. You can also press-and-hold the badge with the mouse for 2 seconds to get a clickable keypad.
- Belt-and-braces: right-click, middle-click, Ctrl+zoom, text selection, drag, back/forward navigation (history trap), and page-close (`beforeunload`) are all blocked or trapped while locked.

### Honest limits

- Full keyboard lock needs a **Chromium browser** (Chrome/Edge). On Firefox/Safari, Esc exits fullscreen immediately — the auto-relock overlay still catches it on the next input, but the lock is weaker.
- No web page can block **Ctrl+Alt+Del** (Windows) or, on some platforms, **Alt+F4**. For absolute lockdown, pair with OS kiosk features (Windows: `chrome --kiosk`, iPad: Guided Access, Android: screen pinning).
- Keyboard lock requires a secure context: `file://`, `https://`, or `localhost` all qualify; plain `http://` on a LAN does not.

## The game

- **Letters** pop up as giant colorful glyphs with particle bursts; an optional voice speaks each letter.
- **Numbers** show the digit plus that many little emoji (3 → 🐟🐟🐟).
- **Space/Enter** fire full-screen confetti; other keys pop themed emoji.
- Every key plays a note on a **pentatonic scale** (same key → same note), so smashing always sounds musical — all synthesized live with WebAudio, zero audio files.
- **6 themes** (Space, Ocean, Garden, Candy, Rescue, Builder), ambient drifting backgrounds, mouse sparkle trails, a combo meter with celebration milestones, and a keys-smashed counter.
- On unlock, parents get a **Smash Report**: total presses, minutes played, favorite keys, max combo, and a chaos-level title — plus settings (sound, voice, theme, reduced motion).

## Hosting (optional)

It's one static file. Drop it on GitHub Pages, Netlify, Cloudflare Pages, or any static host — or don't host it at all and just open the file.

## Development

Everything lives in `index.html` (CSS + three JS modules: `TF.audio`, `TF.engine`, and the shell/lock state machine). `SPEC.md` documents the architecture, module APIs, and the lock state machine.
