# CLAUDE.md — guide for AI agents working on YATD

YATD is a NetHack-flavored ASCII tower defense game. **The entire game is
one file: `index.html`** — HTML, CSS (CRT effects), and vanilla JS, zero
dependencies, no build step. Keep it that way unless the user says
otherwise. Open the file in a browser to play; serve with
`python3 -m http.server` for testing.

`ROADMAP.md` lists planned enhancements in rough priority order.

## Architecture of index.html

The `<script>` is organized top-to-bottom in sections (marked with
`/* ---------- section ---------- */` comments):

1. **Terminal grid** — an 80×24 character-cell buffer (`buf`) rendered to
   a `<canvas>` each frame by `blit()`. Row 0 is the message line, rows
   1–21 are the map (`MAPY = 1` offset), rows 22–23 are the status bar.
   Draw with `put(x, y, ch, fg, bg)` and `text(x, y, str, fg)`. The CRT
   look (scanlines, vignette, glow, flicker) is pure CSS on overlays —
   don't add per-cell effects in JS, they're too slow.

2. **Dungeon map** — generated at load from data, NOT hand-drawn strings:
   - `ROOMS`: floor rectangles `[x1,y1,x2,y2]`; walls (`|` `-`) are drawn
     one cell outside the rect.
   - `WAYPOINTS`: the creep path as orthogonal waypoint list. Carving
     turns rock `' '` into corridor `#`, room walls into doors `+`, and
     leaves room floor `.` alone. `path` is the ordered `[x,y]` cell list;
     `pathIdx` maps `"x,y"` → path index.
   - To add a new level: define a new ROOMS/WAYPOINTS pair and re-run the
     carving code (currently it runs once at top level — refactor into a
     `buildLevel(rooms, waypoints)` function first).
   - Placement rules: `buildable(x,y)` = room floor off-path;
     `trapPlaceable(x,y)` = on-path corridor/door. Both exclude occupied.

3. **Data tables** — `TOWERS` (1–4, keyed by hotkey), `MONSTERS` (keyed by
   their map glyph: `r k a o Z T D`), `WAVES` (array of waves; each wave
   is a list of `{t: glyph, n: count, gap: seconds}` groups spawned in
   order). `waveHpScale(w)` scales monster HP per wave. **Balance lives
   entirely in these tables** — tune here, not in the logic.

4. **Game state** — the global `G` object (also `window.G` for tests).
   `G.state` machine: `title → build ⇄ wave → won | lost`. Waves are
   pre-expanded into `G.spawnQueue` (`{at, t}` sorted by time) by
   `startWave()`. Creeps store `prog`, a float index into `path`;
   `creepCell(c)` gives their cell. No pathfinding exists or is needed —
   the path is fixed.

5. **Combat** — `updateTowers(dt)` handles all four tower kinds (`bolt`,
   `splash`, `slow`, `trap`); targeting is "furthest along path in
   Euclidean range". `damageCreep()` handles death, bounty, floats.
   Visuals are `G.fx` (short-lived beam/splash glyph overlays) and
   `G.floats` (rising `+3$` texts).

6. **Rendering** — `drawGame()` rebuilds the whole cell buffer every
   frame: map → range highlight → towers → creeps → fx → cursor →
   message/status lines → optional help overlay. Title/won/lost have their
   own draw functions. Full redraw is cheap; don't add dirty-tracking.

7. **Input** — mouse (hover + click) and keyboard (vi-keys cursor,
   hotkeys) both funnel into `activate(x, y)`. Add new hotkeys in the
   single `keydown` switch. Note: `u` upgrades a selected tower but falls
   through to nothing otherwise; `h j k l b n y` are cursor movement, so
   avoid reusing those letters.

8. **Main loop** — fixed-timestep (`STEP = 1/60`) accumulator inside
   `requestAnimationFrame`; `G.speed` multiplies accumulated time. Put
   all gameplay logic in `update(dt)`, never in draw functions.

## Testing (do this before every push)

`window.G` (live state) and `window.YATD` (API: `buildTower`,
`upgradeTower`, `sellTower`, `startWave`, `resetGame`, `buildable`,
`trapPlaceable`, `towerAt`, `path`, `map`) are exposed as test hooks —
keep them working.

Verify with headless Chromium (Playwright). Pattern that works in the
Claude Code cloud environment:

```js
chromium.launch({ executablePath: '/opt/pw-browsers/chromium' })
```

1. Load the page, check for zero page/console errors, screenshot the
   title, mid-wave, victory, and tombstone screens and LOOK at them.
2. Sanity-check via `page.evaluate`: building deducts gold, illegal
   placement refused, traps only on path, selling refunds 70%.
3. Run a full automated playthrough at `G.speed = 4`: greedily place
   towers on tiles ranked by path coverage each build phase, start waves,
   assert final state is `won`. **The game must stay winnable by a good
   strategy after any balance change.** If your bot wins with zero HP
   lost, difficulty is fine or could go up; if it loses, tune down.

## Conventions

- NetHack voice everywhere: messages read like NetHack ("You feel a
  draft...", zorkmids for gold, tombstone on death). Monsters are letter
  glyphs with classic meanings; keep new ones consistent (`;` eel, `&`
  demon, `L` lich...).
- Colors: phosphor-green baseline (`#3d6b3d` floors, `#50ff96` accents);
  monsters/towers get muted CRT-friendly colors, full white reserved for
  hit-flash.
- No external assets: no CDNs, fonts, images, or fetches. The strict
  one-file constraint is the point of the project.
- Git: work on `main` and push there (user's standing instruction).
