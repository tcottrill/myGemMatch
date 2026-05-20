# My Gem Match

A mobile-friendly Bejeweled-style match-3 puzzle — swap adjacent gems to line up three or more of the same colour, watch the cascades fly, and chase a high score. Built as a single self-contained `index.html` with no build step, no server, and no network calls after the first page load. The whole gem set, special-gem effects, and UI chrome are embedded inline.

**Play it now:** [https://tcottrill.github.io/myGemMatch/](https://tcottrill.github.io/myGemMatch/)

Seven jewel colours, three power gems (fire, star, hyper) with proper Bejeweled-2 spawn and activation rules, cascading combos with a score multiplier, animated particle bursts and floating score pops on every match, a soft idle "breath" so the board feels alive, hint highlights when you idle, deadlock-aware auto-shuffle, synthesized sound effects, and a glowing dark theme. It is also a Progressive Web App (PWA), so you can pin it to your iPhone, Android, or desktop and play it like a native app, full-screen and offline.

## Free and open

This project is released into the **public domain** under the [Unlicense](LICENSE). It is **100% free**:

- No purchase, subscription, or in-app payment.
- No ads, no trackers, no analytics.
- No account or sign-in.
- No network calls — once the page is loaded, the game runs entirely on your device.

You may copy, modify, redistribute, or sell it for any purpose.

## How to play

Bejeweled-style match-3 is a swap-and-cascade puzzle. The 8×8 board is filled with coloured gems; your job is to score points by lining up runs of identical gems.

- **Tap a gem** to select it. A soft gold highlight pulses around it.
- **Tap an orthogonally-adjacent gem** to swap the two. You can also drag/swipe one gem onto its neighbour.
- A swap only commits if it creates a run of **three or more** of the same colour in a row or column. Otherwise the gems animate back to where they started.
- Cleared gems are replaced by ones falling in from above. New matches that form during the refill trigger **cascades** — each chain multiplies your score by 1.5×, and a "COMBO ×N!" banner pops on the board.
- If you idle for about 5 seconds, the game pulses a hint on a known-good swap.

### Power gems

Make a big match in a single swap and you spawn a power gem at the swap-origin cell. Later, match or swap *that* gem to trigger its effect.

| Spawn | How it spawns | When matched / swapped, it… |
| --- | --- | --- |
| 🔥 **Fire gem** | Match **4** in a single line | …clears its 3×3 neighbourhood and shakes the board |
| ✨ **Star gem** | Match **5+** in a single line | …clears its full row **and** column with a screen flash |
| 🌈 **Hyper cube** | Match in a **T** or **L** shape (two runs sharing a cell) | …when swapped with any coloured gem, removes **every** gem of that colour from the board |

Hyper + hyper = the whole board clears in a single super-nova flash.

### Score multipliers

| Match | Base score |
| --- | --- |
| Three in a row | 30 |
| Four in a row | 60 (and spawn a fire gem) |
| Five or more | 100 (and spawn a star gem) |
| Cascade chain N | ×1.5<sup>N−1</sup> on top of base |

Best score is saved in `localStorage` and shown on the menu.

### In-game controls

| Button | What it does |
| --- | --- |
| **☰ Menu** | Mid-game, asks "New Game?" with a confirm. Otherwise opens the main menu directly. |
| **🔊 / 🔇 Mute** | Toggle sound effects. Persists across visits. |

A tap on a gem plays a soft click; a legal swap plays a rising blip; matches play a chord whose pitch climbs with each cascade; big matches play a brighter chord; power-gem activations sweep into a sawtooth flourish; illegal swaps buzz softly.

### Deadlock handling

After every settle, the game scans the board for any legal swap that would create a match. If there are none, a brief "Shuffling…" toast appears and the board is reshuffled into a new playable layout, preserving your score. You'll never sit there with no legal moves.

## Install on your iPhone (pin to home screen)

Because it is a PWA, you can add the game to your home screen and launch it like any other app — full-screen, no Safari address bar.

1. On your iPhone, open **Safari** (this only works in Safari — not Chrome or Firefox on iOS) and navigate to [https://tcottrill.github.io/myGemMatch/](https://tcottrill.github.io/myGemMatch/).
2. Tap the **Share** button (the square with the up arrow) at the bottom of the screen.
3. Scroll down and tap **Add to Home Screen**.
4. Confirm the name (it will default to "Bejeweled") and tap **Add** in the top-right.

A gem icon now appears on your home screen. Tap it to launch the game full-screen — it will look and feel like a native app, with no browser chrome.

### Notes (iPhone)

- iOS requires the page be loaded over **HTTPS** for the home-screen app to launch in standalone mode. The GitHub Pages URL above is HTTPS, so it works out of the box.
- The icon and splash background use the `apple-touch-icon.png` and theme colour already configured in `index.html` and `site.webmanifest`.
- To remove the app, long-press the icon on the home screen and choose **Remove App → Delete Bookmark**.

## Install on your Android phone (pin to home screen)

Yes — Android supports the same PWA install flow, and on Android it produces an even more app-like result: the game installs through the system's WebAPK mechanism, gets its own entry in the app drawer, and runs in its own window without any browser UI.

### Chrome (recommended)

1. Open **Chrome** on your Android phone and go to [https://tcottrill.github.io/myGemMatch/](https://tcottrill.github.io/myGemMatch/).
2. You may see an "Install app" or "Add to Home screen" prompt at the bottom of the screen — tap it and confirm.
3. If no prompt appears, tap the **⋮** (three-dot menu) in the top-right, then choose **Install app** (or **Add to Home screen** on older versions of Chrome).
4. Confirm the name and tap **Install** / **Add**.

A gem icon will appear on your home screen and in your app drawer. Tap it to launch the game full-screen.

### Other Android browsers

- **Samsung Internet:** menu → **Add page to** → **Home screen**.
- **Firefox:** menu → **Install** (or **Add to Home screen**), depending on version.
- **Edge:** menu → **Add to phone**.

### Notes (Android)

- To uninstall, long-press the icon and tap **Uninstall** (or drag it to the **Uninstall** target at the top of the screen) — same as any other app.
- Because Android installs it as a WebAPK, the game will also appear under **Settings → Apps**.

## Install on desktop

In Chrome or Edge, the install icon appears in the address bar when the page is loaded over HTTPS. Click it, or open the **⋮** menu and choose **Install Bejeweled…**. The game gets its own window with no browser chrome and an entry in your Start menu / Dock.

## Run locally

Open `index.html` directly in any modern browser. That's it — the whole game is self-contained, so double-clicking the file works.

To test the PWA install flow on desktop, serve the folder over HTTP — for example:

```sh
# Python 3
python -m http.server 8000
```

Then visit `http://localhost:8000` in your browser.

## How the assets work

`index.html` is a single ~50 KB file that contains:

1. **8 inline SVG gem symbols** as hidden children of a `#gem-sprite` block — the seven colour gems (red, orange, yellow, green, blue, purple, white) plus the `tile_hyper` rainbow cube. Each carries baked-in radial gradient highlights so the polygons read as glossy jewels rather than flat shapes. On boot, each symbol is wrapped as a standalone SVG document, encoded as a `data:image/svg+xml` URL, and loaded as an `Image` for fast canvas blits.
2. **Canvas-side effects** — animated power-gem overlays (pulsing fire glow, rotating star cross, hue-shifting hyper outline), colour-matched particle bursts on every clear, floating "+score" pops, an idle breathing wave, a "COMBO ×N!" banner for cascades, plus a screen flash and shake on big matches.
3. **The full game** — HTML, CSS, JavaScript — including match detection, gravity / refill, power-gem spawn and activation rules, deadlock-aware auto-shuffle, hint system, WebAudio synth, and persistence.

No `fetch`, no XHR, no service workers required. The page loads in well under a second even from disk.

The original gem SVGs come from a `bejeweled.html` source file (kept in the repo for reference); their symbol defs were lifted, namespaced, and enriched with highlights for the build that ships in `index.html`.

## Files

- `index.html` — the entire game (markup, CSS, JS, embedded gem sprites, effects).
- `bejeweled.html` — original gem SVG source, kept as a regeneration reference.
- `site.webmanifest` — PWA manifest (name, icons, theme colour).
- `apple-touch-icon.png`, `apple-touch-icon-152x152.png`, `apple-touch-icon-167x167.png`, `icon-192.png`, `icon-512.png`, `favicon-*.png`, `favicon.ico` — app and tab icons.
- `icon-master.svg` — vector source for the icons.
- `docs/superpowers/specs/` and `docs/superpowers/plans/` — the brainstorming spec and implementation plan that drove the build.
- `LICENSE` — public domain dedication.

## Development

The entire codebase — every line of HTML, CSS, and JavaScript in `index.html`, plus all documentation in this repository — was generated by [Claude](https://claude.ai), Anthropic's AI assistant, over several iterations of prompting, refinement, and play-testing. A human directed the work, made the design and gameplay decisions, and verified the result; Claude wrote the code.

## Credits

- Gem artwork: gem SVG symbols lifted from a provided `bejeweled.html` source.
- Code & docs: generated by [Claude](https://claude.ai) (Anthropic).

## License

[Unlicense](LICENSE) — public domain.
