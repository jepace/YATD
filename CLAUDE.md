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

2. **Dungeon map** — PROCEDURALLY GENERATED, new every game (roguelike!):
   - `genPath()` makes serpentine orthogonal waypoints (x-columns strictly
     increase, so the path can never self-cross); `genRooms(cells)` seeds
     room rects on random path cells, rejecting overlaps (2-cell margin).
   - `buildLevel()` carves into `map`/`path`/`pathIdx` (all mutated IN
     PLACE so `window.YATD` references stay valid): rock `' '` → corridor
     `#`, room walls → doors `+`, room floor `.` untouched. Returns a
     quality score; `genLevel()` retries (seed+i) until score ≥ 60
     buildable tiles near the path AND path length ≥ 135.
   - RNG: `mulberry32(SEED)`, `rnd()`/`ri(a,b)`. `?seed=N` URL param
     replays a dungeon (first game only); seed shown on title screen.
     `resetGame()` regenerates level+waves except for the very first game
     (`levelFresh`), so the title backdrop matches game one.
   - `path`/`pathIdx` are **generation-time only** now (room seeding,
     quality scoring, fountain placement) — they do NOT constrain runtime
     tower placement or monster movement; see the pathfinding note in
     section 4. `ENTRANCE`/`EXIT` (the `<`/`>` cells) are the real runtime
     anchors, set by `buildLevel()`.
   - Placement rules: `buildable(x,y)` = any room floor `.` tile, not
     already occupied by a tower or a creep (current cell OR the cell
     it's already committed to stepping into); `trapPlaceable(x,y)` =
     any corridor/door tile, unoccupied by a tower (creeps are fine to
     trap under — traps don't block). Both can now target tiles monsters
     actually walk over — see section 4.

3. **Data tables** — `TOWERS` (1–4, keyed by hotkey; glyphs are always
   NetHack object-class punctuation — `/` `0` `{` `^` — never a letter,
   since letters are reserved for monsters), `MONSTERS` (keyed by
   their map glyph: `r x a b k ; o & Z T D`). Monster flags that interact
   with the pathfinding/combat systems: `flies: true` (bat) makes a
   creep use `G.flowFly` instead of `G.flow` — see the pathfinding note
   below, this is the counter to a turtled tower maze, it ignores
   blocking towers entirely (still stopped by rock/walls); `splits: true`
   + `splitChance`/`splitHpFrac` (demon) spawns a weaker copy of itself
   on death via `spawnChild()`, which sets `child: true` on the copy so
   it can't chain-split; `submerges: true` (eel) makes `isSubmerged(c)`
   (a per-creep `c.subPhase`-offset time cycle, `% 4 < 1.5`) return true
   ~40% of the time, during which it's excluded from every damage
   source's targeting filter (trap victims, bolt/splash/slow targets,
   splash's secondary AoE loop) EXCEPT `useScroll()`'s fireball, which
   intentionally ignores submersion — if you add a new AoE/instant-hit
   effect, decide deliberately whether it should respect submersion and
   say so in a comment, since it's easy to forget one of the four
   existing filter sites when adding a fifth. `WAVES` is regenerated on
   every new level by `genWaves(dlvl)`: each wave spends a point budget
   (~8×1.5^(w−1) × 1.3^(dlvl−1), ±15%) on 1–3 monster types from a pool
   gated by minimum wave (`min`); wave 10 always appends the dragon.
   `waveHpScale(w, dlvl)` scales monster HP per wave and per depth — but
   the *per-wave* factor never applies to the boss (`m.boss` → only the
   *per-dlvl* factor, `Math.pow(1.15, dlvl-1)`, computed inline in
   `spawnCreep`), or random short paths make wave 10 unwinnable.
   **Balance lives in these tables and `genWaves`'s budget curve** — tune
   there, not in the logic.
   The Wand of Striking tower (`TOWERS[1]`) has limited `maxCharges`,
   scaling with level via `towerMaxCharges(t)` (×1.5 per level, up to
   `MAX_LVL` = 4) and regenerating passively at `CHARGE_REGEN`
   charges/sec even while off cooldown or mid-wave; enchanting a wand
   (`upgradeTower`) refills it to its new max. A depleted wand renders
   dimmed (`empty` check in the tower draw loop) and simply stops firing
   — `updateTowers` `continue`s past it without resetting `t.cd`, so it
   fires again the instant a charge trickles back in.
   Items: monsters have an 8% drop chance (`dropItem`): gold pile,
   `?` scroll of fireball (`r`, 45 dmg to all), `!` potion of haste
   (`q`, towers 2× rate 8s via `G.hasteUntil`), `/` wand of cold
   (`z`, slow all 4s). Inventory in `G.inv`; view with the `i` overlay
   (`drawInventory`). All game events go through `message(s)`, which
   both sets the message line and appends to `G.log` (viewed via the
   `log` overlay, `Ctrl-P`, NetHack-style). There is no floating combat
   text — kills, drops, leaks etc. are all reported as messages, not
   `G.fx`-style popups.

4. **Game state** — the global `G` object (also `window.G` for tests).
   `G.state` machine: `title → build ⇄ wave → won | lost`. Waves are
   pre-expanded into `G.spawnQueue` (`{at, t}` sorted by time) by
   `startWave()`.

   **Dungeon levels:** clearing wave 10 doesn't end the game until
   `G.dlvl >= MAX_DLVL` (5) — below that, `update()`'s wave-clear branch
   calls `descend()` instead: `G.dlvl++`, a brand new `genLevel()` +
   `genWaves(G.dlvl)`, towers/creeps cleared (new map, old towers make no
   sense on it), `G.flow`/`G.flowFly` recomputed, and a small stairwell
   heal (`+25%` of missing HP) — but gold, score, and the rest of HP
   carry over. Only at `MAX_DLVL`'s wave 10 does the state become `won`.
   If you add anything else that should reset per-level (as opposed to
   per-game), it needs a line in both `resetGame()` AND `descend()` —
   they're intentionally separate functions, not one calling the other,
   because `resetGame()` also resets things (gold, HP, score, inventory)
   that `descend()` must NOT touch.

   **Pathfinding (maze-building TD, not a fixed-path TD):** towers 1–3
   (`TOWERS[n].blocks === true`) physically occupy their tile; monsters
   have no fixed route and instead follow a BFS flow field recomputed
   from scratch by `computeFlow()` every time a blocking tower is built
   or sold (`G.flow = computeFlow()`, stored on `G`, `{dist, next}`
   indexed `[y][x]`) — `dist` is steps-to-exit, `next` is which
   neighboring cell to step toward. `terrainWalkable(x,y)` (rock/walls
   only excluded — corridors, rooms, doors, decorative fountains are all
   walkable) defines the graph; blocking towers are the only dynamic
   obstacles. `buildTower()` calls `wouldSealPath(x,y)` (a `computeFlow`
   dry-run with the candidate cell pre-blocked) before committing to a
   *blocking* tower placement — if the entrance would become unreachable
   from the exit, the build is refused with a message; this is the only
   thing stopping a player from walling the stairs shut. The fire trap
   (`blocks: false`) never triggers this check — it's flush with the
   floor, monsters walk over it, `trapPlaceable` doesn't care about
   connectivity.

   `computeFlow(extraBlocked, ignoreTowerBlocks)`'s second argument
   produces `G.flowFly` — computed once per level in `resetGame()` (not
   recomputed on every build/sell, since it's static: flying creeps
   ignore every tower, so it only ever depends on the fixed rock/wall
   layout). `creepFlow(c)` picks `G.flowFly` vs `G.flow` per creep based
   on `MONSTERS[c.type].flies`; both `advanceTarget()` and
   `creepProgress()` go through it so a flying creep is fully consistent
   end to end (movement, targeting priority) without touching any other
   code path.

   Creeps move cell-to-cell: `c.x,c.y` = current cell, `c.nx,c.ny` = the
   cell they're already committed to stepping into (from
   `advanceTarget(c)`, looked up fresh from `G.flow.next` every time they
   finish a step — this is how they react live to a tower appearing
   mid-wave), `c.t` = progress toward that step in `[0,1)`, incremented
   by `spd*dt` in `updateCreeps` and consumed via a `while (c.t >= 1)`
   loop (handles multiple cells crossed in one frame at high speed/game
   speed). A creep is considered to have LEAKED the instant it steps onto
   `EXIT`. Creeps also carry a per-creep `spd` variance (±15%) and a
   persistent `jit` lane (-1/0/1) so they don't all march in lockstep.
   `creepCell(c)` is the single source of truth for a creep's actual
   cell — it applies `jit` as a perpendicular offset from the creep's
   current travel direction (`c.nx-c.x, c.ny-c.y`) ONLY when the base
   cell is room floor `.` (never in 1-wide corridors, so trap alignment
   and targeting stay exact); it's used for everything: rendering, tower
   targeting, trap collision, and leak/hover checks. `creepProgress(c)`
   (flow distance minus `c.t`) replaces the old "furthest along the
   path" sort key for tower targeting — lower is closer to the exit.

   **If you touch this system:** any change to `buildable`/`trapPlaceable`
   or to when `G.flow` gets recomputed needs to preserve the invariant
   that `wouldSealPath` is checked before every *blocking* placement —
   skipping it lets the player permanently softlock a wave (monsters
   spawn with no route and `advanceTarget` returns `nx===x,ny===y`,
   which `updateCreeps` handles by having them idle in place rather than
   crashing, but that's a bug state, not a feature).

5. **Combat** — `updateTowers(dt)` handles all four tower kinds (`bolt`,
   `splash`, `slow`, `trap`); targeting is "furthest along path in
   Euclidean range AND line of sight" — `hasLineOfSight(x0,y0,x1,y1)`
   (Bresenham walk, blocked by `" "`/`"|"`/`"-"` cells) gates every
   non-trap tower so they can't shoot through walls into an adjacent
   corridor; splash's secondary AoE victims are also LOS-checked from the
   impact point, not just the primary target from the tower. Traps don't
   need LOS (contact-triggered). `damageCreep()` handles death, bounty,
   and messages. Visuals are `G.fx` only (short-lived beam/splash glyph
   overlays) — no floating combat text; all of that is `message()` calls
   now.

6. **Rendering** — `drawGame()` rebuilds the whole cell buffer every
   frame: map → range highlight → towers → creeps → fx → cursor →
   message/status lines → optional overlay. Exactly one of three overlays
   can be open at a time via `G.overlay` (`null | "help" | "inv" | "log"`),
   drawn last so it sits on top: `drawHelp()`, `drawInventory()`,
   `drawLog()`, all built on the shared `overlayBox(x0,y0,w,h,title)`
   helper. Keep every overlay's box within rows 1–20 (rows 22–23 are the
   status bar; row 21 is the last map row — don't let `y0 + h` reach 22 or
   you'll paint over the status text drawn earlier that frame). Title/won/
   lost have their own draw functions. Full redraw is cheap; don't add
   dirty-tracking.

7. **Input** — mouse (hover + click) and keyboard (vi-keys cursor,
   hotkeys) both funnel into `activate(x, y)`. Add new hotkeys in the
   single `keydown` switch. Note: `u` upgrades a selected tower but falls
   through to nothing otherwise; `h j k l b n y` are cursor movement, so
   avoid reusing those letters. `?`, `i`, and `Ctrl-P` toggle the three
   overlays (see above) from anywhere and are checked before the overlay
   swallows other keys — add new overlay-toggle keys next to them, not
   inside the `switch`. Tower placement is intentionally NOT sticky:
   `G.selType` (the readied tower, 0 = none) is cleared by `activate()`
   after every successful `buildTower()`, so the player must press 1–4
   again before each placement — this prevents accidental multi-builds
   when clicking around or moving the keyboard cursor. `SPACE` is
   contextual: `startWave()` while `G.state === "build"`, otherwise
   toggles `G.paused`. Bindings intentionally mirror NetHack where one
   exists (`r`ead, `q`uaff, `z`ap, `i`nventory, `Ctrl-P` message log).

8. **Main loop** — fixed-timestep (`STEP = 1/60`) accumulator inside
   `requestAnimationFrame`; `G.speed` multiplies accumulated time. Put
   all gameplay logic in `update(dt)`, never in draw functions.

## Testing

**Default: don't launch a browser to click through the game.** The user
has asked to keep this to code review, not play-testing — verify changes
by reading the diff carefully (trace the affected function calls by
hand, check for off-by-ones, confirm data-table edits match the format
of surrounding entries) and running a syntax check:

```bash
python3 -c "import re; open('/tmp/yatd.js','w').write(re.search(r'<script>(.*)</script>', open('index.html').read(), re.S).group(1))"
node --check /tmp/yatd.js
```

That's sufficient for most changes (data-table tweaks, new hotkeys, UI
text, glyph/color changes, refactors). Reserve full headless-browser
verification for when the user explicitly asks for it, or for changes
where static reading genuinely can't tell you the outcome (new physics/
collision math, procedural generation quality, anything touching win/
loss balance). When that's asked for:

`window.G` (live state, including `G.flow`/`G.flowFly`) and `window.YATD`
(API: `buildTower`, `upgradeTower`, `sellTower`, `startWave`, `resetGame`,
`buildable`, `trapPlaceable`, `towerAt`, `path`, `map`, `wouldSealPath`,
`getEntrance()`, `getExit()`, `getFlow()`, `getFlowFly()`) are exposed as
test hooks — keep them working. `getEntrance`/`getExit`/`getFlow`/
`getFlowFly` are functions, not plain properties, because
`ENTRANCE`/`EXIT`/`G.flow`/`G.flowFly` are wholesale *reassigned* (not
mutated) on every `resetGame()` — a plain property snapshot at export
time would go stale after the first game. Verify with headless Chromium
(Playwright); this pattern works in the Claude Code cloud environment:

```js
chromium.launch({ executablePath: '/opt/pw-browsers/chromium' })
```

1. Load the page, check for zero page/console errors, screenshot the
   title, mid-wave, victory, and tombstone screens and LOOK at them.
2. Sanity-check via `page.evaluate`: building deducts gold, illegal
   placement refused, traps only on corridor/door tiles, selling refunds
   70%. Also check the maze invariant directly: pick a cell adjacent to
   the entrance, confirm `YATD.wouldSealPath(x,y)` agrees with whether
   `YATD.buildTower(1,x,y)` actually succeeds, and confirm
   `YATD.getFlow().dist` at the entrance stays finite after any
   sequence of builds you make in the test.
3. Run a full automated playthrough at `G.speed = 4`: greedily place
   towers on the highest-value tiles each build phase (e.g. rank by
   proximity to `YATD.getEntrance()`/`getExit()` or just try tiles in
   the rooms — since the route is now dynamic there's no fixed "path
   coverage" to rank against), start waves, assert final state is `won`.
   **A full win now means clearing 5 dungeon levels (50 waves total,
   `MAX_DLVL`), which takes a while even at 4x speed — budget for it, or
   for a quicker smoke test assert `G.dlvl` reaches 2 (confirms
   `descend()` fires and the new level is playable) instead of chasing a
   full win every time.** Because maps and waves are random, run at
   least 3 playthroughs (each `resetGame()` = new dungeon) and require
   all to reach the bar you picked. If every run clears with zero HP
   lost, difficulty is fine or could go up; if any loses, tune down.
4. Also assert: two `resetGame()` calls produce different maps/seeds,
   loading with `?seed=N` reproduces a map, `descend()` clears
   `G.towers`/`G.creeps` and leaves `G.gold`/`G.score` untouched, and —
   given the maze mechanic is new and easy to get subtly wrong — that a
   creep never gets stuck idling for an entire wave (poll `G.creeps` for
   any whose `x,y` hasn't changed across several seconds of wall-clock/
   game time while `G.state === "wave"`).

## Conventions

- NetHack voice everywhere: messages read like NetHack ("You feel a
  draft...", zorkmids for gold, tombstone on death). Monsters are letter
  glyphs with classic meanings; keep new ones consistent with real
  NetHack (`L` lich is a good next pick). `; x &` are already taken
  (eel, grid bug, demon — see section 3).
- Colors: phosphor-green baseline (`#3d6b3d` floors, `#50ff96` accents);
  monsters/towers get muted CRT-friendly colors, full white reserved for
  hit-flash.
- No external assets: no CDNs, fonts, images, or fetches. The strict
  one-file constraint is the point of the project.
- Git: work on `main` and push there (user's standing instruction).
