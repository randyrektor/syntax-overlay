# AGENTS.md

Conventions for working in this repo. Applies to humans and AI agents alike.

## Project shape

- Plain static HTML / CSS / vanilla JS. **No build step, no bundler, no framework, no npm install.** A page is one self-contained file.
- Each overlay file inlines its own `<style>` and `<script>`. We trade some duplication for zero tooling overhead and "open the file in a browser" simplicity.
- Read [`README.md`](./README.md) before changing how files talk to each other; the WebSocket envelope and channel numbers are a contract.

## File layout

```
admin.html               Tabbed operator command-center
featured-comment.html    SSN featured-chat OBS overlay (read-only)
lower-thirds.html        Lower-thirds operator panel + OBS overlay
ticker.html              News-ticker operator panel + OBS overlay
assets/lower-thirds/     Host avatar SVGs (360×360 canvas, face centered)
README.md                Architecture + how to run + OBS setup
AGENTS.md                You are here
```

Don't introduce new top-level directories without a reason. Don't add `node_modules`, `dist`, `build`, etc.

## Temporary / scratch files

Anything you create just to verify something during a task — test scripts, screenshots, generated samples — goes in `_work-tmp/`. That folder is intentionally not tracked; treat it as ephemeral.

## JavaScript style

- ES5-flavored vanilla JS inside an IIFE: `(function(){ 'use strict'; … })()`. We use `var`, function declarations, no arrow functions, no `let`/`const`. This keeps the code uniform across files and avoids any temptation to reach for a transpiler.
- DOM API directly, no jQuery, no querySelector helpers.
- Defensive: every external input goes through `esc()` before being interpolated into HTML; URLs through `escUrl()` (in `featured-comment.html`); colors through `safeCssColor()`.
- Section markers: `/* ── SECTION NAME ── */`. Keep them — they make the long single-file scripts navigable.

## CSS style

- Single inline `<style>` block per file, top-down by section (RESET → LAYOUT → COMPONENT → ANIMATIONS → STATES).
- Match the existing color palette and the chunky stepped box-shadow signature (see README for tokens). Don't introduce new accent colors without coordinating.
- `Space Grotesk` for body text, `Space Mono` for labels / chrome / buttons.

## Comments

- Comment **why**, not **what**. The code already says what.
- Document non-obvious things: protocol contracts, race conditions, browser quirks (Chromium / OBS CEF), specificity tricks, intentional layout offsets.
- Don't leave conversational comments ("we just changed", "fixes the issue from earlier"). Future readers don't have the conversation.
- Section markers are the exception — they're navigation, not narration.

## Sync protocol

Two-way overlays (lower-thirds, ticker, anything new) follow the envelope and channel scheme in `README.md`. **Don't reuse a channel number across tools** even though the namespacing key would prevent collisions — separate channels keep each tool's traffic isolated and easy to debug in network tools.

When adding a new tool:
1. Allocate the next unused channel number.
2. Pick a unique `<toolKey>` for the envelope (`syntax<Thing>` naming).
3. Update the channel/key table in `README.md`.

## URL parameter conventions

Every overlay accepts:
- `?session=<id>` (aliases: `?s=`, `?room=`) — the SSN session room
- `?obs` or `?overlay` — render as a transparent OBS Browser Source (no controls)
- `?localserver` — point WebSocket at `ws://127.0.0.1:3000` for local SSN dev
- `?server=<url>` — explicit WebSocket override

When introducing a new param, prefer matching this pattern.

## Local development

```bash
python3 -m http.server 8765
# open http://localhost:8765/admin.html
```

Or any equivalent static server. Browsers cache aggressively, so use a hard reload (Cmd+Shift+R) after edits — and in OBS, refresh the Browser Source from its right-click menu.

## Testing

There is no automated test suite. The smoke-test loop is:
1. Start a local static server.
2. Open `admin.html?session=<test-room>` in one tab and `<tool>.html?session=<test-room>&obs` in another (or in OBS).
3. Trigger an action in the operator tab and confirm the overlay tab/OBS reflects it within a beat.

If you change the sync envelope, run this against every tool that uses it.

## Pull request checklist

- [ ] Comments still explain *why*, not *what*.
- [ ] No `console.log`s, debugger statements, or commented-out code.
- [ ] No new top-level dependencies added.
- [ ] If you touched the sync envelope, channels, or session params, the README table is updated.
- [ ] Smoke-tested in a real OBS Browser Source (operator → overlay round-trip), not just a normal browser tab.
