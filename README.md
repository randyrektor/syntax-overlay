# Syntax Overlay

Browser-based stream overlays for Syntax.fm livestreams. Plain HTML / CSS / vanilla JS, no build step. Each overlay is a single self-contained file you can open in a browser or load into an OBS Browser Source.

## Files

| File | Purpose |
| --- | --- |
| `admin.html` | Operator command-center. Tabs into every operator panel and one-click copies OBS Browser Source URLs. |
| `featured-comment.html` | OBS overlay for [Social Stream Ninja](https://socialstream.ninja) featured chat messages. Read-only — operators interact via SSN's own dock UI. |
| `lower-thirds.html` | Operator panel + OBS overlay for hosts' lower thirds. Same file in two modes (`?obs` strips the controls). |
| `ticker.html` | Operator panel + OBS overlay for the news-style scrolling ticker. Same file in two modes (`?obs` strips the controls). |
| `assets/lower-thirds/*.svg` | Host avatars (CJ, Scott, Wes, Randy). All authored on a 360×360 canvas with the face in the same position so one set of CSS values works for every host. |

## Run it

Any static server works. From the repo root:

```bash
python3 -m http.server 8765
# then open http://localhost:8765/admin.html
```

For production, host the directory anywhere static (GitHub Pages, Netlify, Cloudflare Pages, S3, etc.). There are no dependencies, no env vars, and no server-side code.

## OBS setup

For each overlay, add a **Browser Source** in OBS pointing at the appropriate URL:

| Overlay | URL pattern |
| --- | --- |
| Featured chat | `https://your-host/featured-comment.html?session=<ROOM>` |
| Lower thirds | `https://your-host/lower-thirds.html?session=<ROOM>&obs` |
| Ticker | `https://your-host/ticker.html?session=<ROOM>&obs` |

The admin panel's **Copy OBS URL** button generates the right URL for the active tab.

**Browser Source settings that matter:**
- **Use custom frame rate: 60** — the marquee/scroll animations are 60 fps, OBS defaults to 30 and will visibly stutter.
- Settings → Advanced → **Browser source hardware acceleration: on** (requires an OBS restart).
- Width / Height should match your canvas resolution; let OBS scale rather than the page.

## Sessions

Every operator panel and OBS overlay joins a "session room" on the Social Stream Ninja relay. Pages with the same `?session=<id>` see each other; pages with different sessions don't. Pick any string — operators and OBS just need to use the same one.

`admin.html?session=<id>` propagates the session into both embedded operator panels automatically.

## Architecture

### Two transport channels

- **Featured comment overlay** is a passive consumer. It joins SSN's chat relay and renders whatever the SSN dock sends. The contract (channel numbers, payload shape, clear semantics) is documented at the top of `featured-comment.html`.
- **Lower thirds + ticker overlays** are bidirectional operator tools. They reuse the SSN relay as a generic peer-to-peer pubsub on dedicated channels. Operator pages and `?obs` overlays are the same code in different modes.

### Sync envelope (lower thirds + ticker)

Every payload follows the same shape so the three overlays can share a session room without colliding:

```json
{
  "overlayNinja": {
    "<toolKey>": {
      "action": "<actionName>",
      "_tab": "<unique-tab-id>",
      "...": "tool-specific fields"
    }
  }
}
```

| Tool | `<toolKey>` | SSN channel (in/out) |
| --- | --- | --- |
| Lower thirds | `syntaxLT` | 7 |
| Ticker | `syntaxTicker` | 8 |
| _(reserved for chat)_ | _none — uses SSN's own envelope_ | 1, 2, 3 |

`_tab` lets each tab filter out the relay's echo of its own broadcasts.

### State management pattern (ticker reference implementation)

The ticker is the cleanest example of the pattern future tools should follow:

1. State is split into **replicated** (synced) and **local** (UI-only) buckets.
2. User actions call `applyState(patch)` which mutates state, persists to `localStorage`, re-renders, and broadcasts a full snapshot.
3. Incoming sync messages call `applyState(patch, { silent: true })` — same path, no rebroadcast (this is what prevents echo loops).
4. On WebSocket open: send `state-request`. If you receive one and have non-empty state, respond with a snapshot.
5. Conflict resolution is "last writer wins" — sufficient for a small operator team. If you need stronger ordering, add a logical clock to the snapshot.

## Adding a new overlay tool

1. Create `<tool>.html` modeled on `ticker.html`. Self-contained: operator UI in normal mode, OBS overlay when `?obs` is in the URL.
2. Pick an unused SSN channel number (currently 1, 2, 3 belong to SSN chat; 7 = lower thirds; 8 = ticker; pick 9+).
3. Pick a unique `<toolKey>` for your sync envelope (e.g. `syntaxPoll`).
4. Add it to the `TABS` map in `admin.html` and add a matching `<button class="tab">` and `<iframe>` to the markup.
5. Document the channel + envelope key in this README's table.

Visual conventions to keep things on-brand:
- Fonts: `Space Grotesk` for body text, `Space Mono` for labels and chrome.
- Color tokens (used across all files): bg `#0e0e0d`, card `#141413`, border `#454441`, accent `#ffd54a`, success `#3dd816`, danger `#c43d3c`, text `#f7f6f2`, muted `#8a8883`.
- The chunky stepped 1px box-shadow `1px 1px 0 0 #050504, 2px 2px 0 0 #050504, …` is the signature card shadow — reuse it on any new card-like surface.
- All overlays support `?obs` (or `?overlay`) to render with a transparent background and no operator chrome.
- `?session=<id>`, `?s=<id>`, `?room=<id>` are all accepted as session aliases.

## Coding conventions

See [`AGENTS.md`](./AGENTS.md).

## License

Internal Syntax.fm tooling. No upstream license declared.
