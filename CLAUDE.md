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
     increase, so the path can never self-cross); `genRooms(cells,
     forceCells)` seeds room rects on random path cells, rejecting
     overlaps (3-cell margin — widened from 2 so two independently-valid
     doors on adjacent rooms don't visually merge into a wider gap than
     either one alone). It also rejects a candidate room outright if any
     of its 4 walls would run flush alongside 2+ *other* path cells
     (`wallHasParallelRun`, checked against the raw path-cell list, not
     the carved map) — a corridor that merely tracks parallel to a wall
     for a stretch would otherwise turn every one of those cells into a
     door in `buildLevel()`'s carving pass, reading as one wide gap
     stacked with redundant doorways rather than a real doorway. Forced
     rooms (see below) can't be rejected outright since they must
     succeed, so they instead re-roll `roomAt`'s random size/offset up to
     20 times looking for a parallel-run-free placement, falling back to
     the last attempt if none turn up clean. `forceCells` (always
     `[wp[0], wp[last]]`, the entrance/exit waypoints) get a room placed
     on them FIRST, unconditionally, via the same `x1`/`y1` formula the
     random rooms use — provably, that formula's clamps only ever move a
     room's origin toward its seed cell, never past it, so the seed cell
     always ends up inside the room's own floor rectangle. This
     guarantees the stairs always land on real room floor, never a bare
     corridor or a wall — before this, `ENTRANCE`/`EXIT` were just
     wherever the first/last path waypoint happened to fall, with no such
     guarantee.
   - `buildLevel()` carves into `map`/`path`/`pathIdx` (all mutated IN
     PLACE so `window.YATD` references stay valid): rock `' '` → corridor
     `#`, room walls → doors `+`, room floor `.` untouched — EXCEPT a
     room's own 4 corner wall cells, which are collected into a `corners`
     `Set` and excluded from door conversion **only when isolated**: a
     corridor that merely grazes a corner in isolation is diagonal-only
     from that room's interior floor (movement is strictly orthogonal,
     see `computeFlow`), so a door there would be a dead end that looks
     connected on screen but isn't reachable. `isolatedCorner(x, y)`
     checks this by requiring NONE of the cell's 4 orthogonal neighbors
     to also be on the path (`cellSet`, the full path cell list, built
     upfront so this check doesn't depend on carving order) — **a corner
     with a path neighbor on either side is load-bearing for the
     corridor's own contiguity and must still become a door**, full stop,
     even though it's a corner. This distinction is the fix for a real
     shipped regression: an earlier version excluded *every* corner
     unconditionally, on the theory that "the path segment that seeded a
     room never hits its own corners" — true, but it ignored that a
     *different*, unrelated corridor segment can legitimately run
     straight through another room's corner as part of its own main
     route (e.g. a vertical corridor flush against a room's side wall,
     crossing that wall's corner exactly where the room's floor ends) —
     severing that isn't discarding a cosmetic dead end, it's cutting the
     corridor in half. That version disconnected roughly 2/3 of generated
     levels entirely (confirmed by a scratch regression test comparing
     both versions across 1000 seeds) before being caught. **If you ever
     touch corner/door logic again, run a same-shape check**: generate
     many levels and assert `computeFlow().dist` at `ENTRANCE` is finite
     with zero towers built — a room merely being non-empty or scoring
     well on the existing quality/path-length gate does NOT imply the
     level is actually connected end to end.
     Returns a quality score; `genLevel()` retries (seed+i) until score ≥
     60 buildable tiles near the path AND path length ≥ 135 (verified this
     invariant doesn't hurt the retry success rate: 200/200 in a scratch
     test after this change, same as before it).
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

3. **Data tables** — `TOWERS` (1–6, keyed by hotkey; glyphs are always
   NetHack object-class punctuation — `/` `0` `{` `^` `\` `%` — never a
   letter, since letters are reserved for monsters), `MONSTERS` (~30
   entries, keyed by a descriptive id string — `rat`, `jackal`, `troll`,
   etc — NOT by glyph; each entry carries its own `ch`, decoupled from
   the key precisely so several tiers can share one letter with a
   different name/color/stats, the same way real NetHack's monster
   classes work — e.g. the canine class `d` covers `jackal` → `wolf` →
   `warg` as the dungeon gets deeper, `o` covers `orc` → `orcapt`, `T`
   covers `troll` → `chieftain`, `&` covers `demon` → `hdevil`. Rendering
   always reads `MONSTERS[c.type].ch`, never the type key itself, so
   adding a same-glyph tier needs zero changes outside the table). Each
   `genWaves(dlvl)` pool entry is gated on two independent axes: `min`
   (which wave *within* the level unlocks it, unchanged from before) and
   `minDlvl`/`maxDlvl` (which dungeon levels it's eligible on at all) —
   without the second axis, a trash-tier monster legal on wave 1 of Dlvl
   1 was equally legal on wave 1 of Dlvl 5, since every level restarts
   its own wave counter at 1. Grid bugs, for instance, cap out at
   `maxDlvl: 2` so they're gone by Dlvl 3; heavier tiers (`chieftain`,
   `kraken`, `lich`, …) carry `minDlvl: 5` so they can't show up early
   and trivialize nothing. **If you add or re-tier a pool entry, keep at
   least one `min: 1` entry uncapped (or capped no lower than `maxDlvl:
   5`)** — `jackal` is that safety net today — because the filter is
   `w >= min && dlvl >= minDlvl && dlvl <= maxDlvl`, and if wave 1 of some
   dlvl ever has zero eligible entries `genWaves` produces an empty
   group for that wave (no crash, just a wave with nothing in it) — the
   `genwaves_test.js`/`flow8_unit_test.js` scratch tests specifically
   assert this never happens across all 50 (dlvl, wave) combinations.
   Monster flags that interact with the pathfinding/combat systems (each
   now spread across 2–4 monsters at different tiers for run-to-run
   variety, not just one signature monster apiece): `flies: true` (bat,
   vampire bat, wraith) makes a
   creep use `G.flowFly` instead of `G.flow` — see the pathfinding note
   below, this is the counter to a turtled tower maze, it ignores
   blocking towers entirely (still stopped by rock/walls); the fire
   trap's victim filter in `updateTowers` also excludes `m.flies` for the
   same reason — a trap is flush with the floor, and a flyer never
   touches the ground it's on, exactly like it never touches a blocking
   tower's tile; `splits: true`
   + `splitChance`/`splitHpFrac` (demon, horned devil) spawns a weaker copy of itself
   on death via `spawnChild()`, which sets `child: true` on the copy so
   it can't chain-split; `submerges: true` (eel, kraken) makes `isSubmerged(c)`
   (a per-creep `c.subPhase`-offset time cycle, `% 4 < 1.5`) return true
   ~40% of the time, during which it's excluded from every damage
   source's targeting filter (trap victims, bolt/splash/slow targets,
   splash's secondary AoE loop) EXCEPT `resolveTargeting()`'s fireball,
   which intentionally ignores submersion — if you add a new AoE/instant-hit
   effect, decide deliberately whether it should respect submersion and
   say so in a comment, since it's easy to forget one of the four
   existing filter sites when adding a fifth. Every monster passively
   heals `BASE_REGEN` (0.4) hp/sec in `updateCreeps` whenever below max
   HP, whether or not it's currently being shot at — `regen` (only the
   troll sets it, at 3) overrides the baseline for a monster that should
   heal distinctly faster. `WAVES` is regenerated on
   every new level by `genWaves(dlvl)`: each wave spends a point budget
   (~8×1.5^(w−1) × 1.35^(dlvl−1), ±15%) on 1–3 monster types from a pool
   gated by minimum wave (`min`); wave 10 always appends the dragon.
   `waveHpScale(w, dlvl)` scales monster HP per wave and per depth — but
   the *per-wave* factor never applies to the boss (`m.boss` → only the
   *per-dlvl* factor, `Math.pow(1.2, dlvl-1)`, computed inline in
   `spawnCreep`), or random short paths make wave 10 unwinnable.
   `towerCost(type)` scales every build/enchant price the same way
   (`Math.pow(1.18, G.dlvl-1)` on top of `TOWERS[type].cost`, so
   `upgradeCost(t)` — which multiplies by `t.lvl+1` — inherits it too):
   the zorkmids you're still holding onto by Dlvl 5 buy less than they
   did on Dlvl 1, which is also *why* a stockpile that felt like plenty
   early on stops feeling that way — it's deliberate, not a bug. `t.spent`
   always records the price actually paid, so `sellTower`'s 70% refund
   naturally reflects whatever dlvl the tower was bought at, no separate
   bookkeeping needed. **Balance lives in these tables, `genWaves`'s
   budget curve, `towerCost`'s dlvl exponent, and `BOUNTY_MULT`** — tune
   there, not in the logic. `BOUNTY_MULT` (currently 0.75) trims both
   gold income and XP together — `damageCreep()`'s kill branch computes
   `bounty = Math.round(m.bounty * BOUNTY_MULT)` once and uses that
   trimmed value for `G.gold`, `G.score`, the kill message, and
   `gainXp()`, rather than the raw `MONSTERS[type].bounty`; it exists as
   a single economy-wide knob rather than hand-editing all ~30 monsters'
   bounty fields, added after a report of finishing Dlvl 1 already at
   max player level with hundreds of unspent zorkmids and every tower
   still at L1 — the early economy was simply too generous to ever feel
   like resources mattered. `PLAYER_XP_THRESHOLDS` (section on player
   level, above) was raised at the same time for the same reason.

   **Tower unlocking** is a second progression axis, independent of
   player level: `TOWERS[1]` (Archer) and `TOWERS[6]` (Dirt Wall) are the
   only types in `G.unlocked` (`{ 1: true, 6: true }`) on a fresh
   `resetGame()` — the cheap, no-refund maze-shaping basics, so a new
   player always has a way to experiment with the maze mechanic even
   before earning any unlock gold; every other type carries
   `unlockCost`, a flat one-time gold payment (NOT dlvl-scaled like
   `towerCost`) that `unlockTower(type)` spends to permanently add it to
   `G.unlocked` — it never re-locks and, like `G.gold`/`G.plevel`,
   survives `descend()` and only resets in `resetGame()`. `menuOptions()`
   dispatches on `G.unlocked[n]` for each buildable-here type: locked
   shows an `Unlock <name> ($cost, one-time)` option (`keepOpen: true`,
   same as upgrade/sell, so the menu naturally shows the real build
   option on its next fresh `menuOptions()` call once you've paid);
   unlocked shows the normal build option. `buildTower()` also checks
   `G.unlocked[type]` directly (defense in depth for the exposed
   `YATD.buildTower` test hook, which bypasses the menu). Player level
   gates how far a tower you already have access to can be enchanted;
   unlocking gates whether you have access to the type at all — don't
   conflate the two when adding a 7th tower type.
   Both charge-limited towers (`TOWERS[1]` Archer, `TOWERS[5]` Wand of
   Lightning) have limited `maxCharges`, scaling with level via
   `towerMaxCharges(t)` (×1.5 per level, up to `MAX_LVL` = 4) and
   regenerating passively at `CHARGE_REGEN` charges/sec even while off
   cooldown or mid-wave; enchanting a wand (`upgradeTower`) refills it to
   its new max. A depleted wand renders dimmed (`empty` check in the
   tower draw loop) and simply stops firing — `updateTowers` `continue`s
   past it without resetting `t.cd`, so it fires again the instant a
   charge trickles back in. (Design note, direct from the user: nothing
   should be perfect and everything should be useful. Concretely this
   means resources ebb and flow rather than being purely one-directional
   — wands drain and recharge, walls take damage and can be repaired,
   monsters take damage and passively regen, recharging is a dice roll
   not a guaranteed top-off — and no single tool should trivialize a
   whole category of problem, which is also why dragon immunity exists.
   If you add a new resource or consumable, default to giving it both
   directions and some randomness/limitation rather than a clean one-way
   or guaranteed effect.)
   `TOWERS[5]` (`kind: "chain"`) is a bolt that arcs: after damaging its
   primary target it looks for the nearest un-hit, non-submerged,
   LOS-visible creep within `chainRange` of the *previous* hit (not the
   tower), deals `chainFalloff`-reduced damage, and repeats up to
   `chainCount` extra times — each hop draws its own `beamFx` segment;
   its charge gate/decrement (`spec.maxCharges && ...`) sits at the top
   of this branch, mirroring the bolt branch's. `TOWERS[6]` (Dirt Wall,
   `kind: "wall"`) is pure obstacle: it
   has `hp`/`towerMaxHp(t)` instead of `dmg`/`range`, `updateTowers`
   `continue`s past it immediately (no combat branch applies), and a
   separate `updateWalls(dt)` (called right after `updateCreeps` in
   `update()`) chips `WALL_DPS` off any wall with a live, non-dead creep
   orthogonally adjacent — reaching 0 hp removes the tower with no
   refund and triggers one `G.flow = computeFlow()`. Both new tower
   kinds needed zero changes to `buildable`/`wouldSealPath`/`computeFlow`
   — they participate in the maze system automatically via `blocks: true`,
   which is the whole point of that flag being generic.
   Items: monsters have an 8% drop chance (`dropItem`): gold pile,
   `?` scroll of fireball (`r`, 45 fire dmg — respects dragon fire
   immunity, see below), `!` potion of haste (`q`, towers 2× rate 8s via
   `G.hasteUntil`), `/` wand of cold (`z`, slows), `/` wand of
   charging (`c`, restores `t.lvl` dice of `d6` to every tower's
   `charges`/`hp` that's below max — deliberately NOT a guaranteed
   full top-off, in keeping with "nothing is perfect" — applies uniformly
   to any tower with `spec.maxCharges` or `spec.hp` set, so the wand of
   lightning benefits from it exactly like the wand of striking with no
   special-casing needed). The fireball scroll and cold wand and charging
   wand are **all targeted, not instant, and not blanket effects** —
   nothing in this game should apply to everything at once (the same
   "nothing is perfect, everything is useful" philosophy, applied to
   *scope* rather than magnitude): `r`/`z`/`c` calls
   `beginScroll()`/`beginWand()`/`beginCharge()`, which arms
   `G.targeting = { kind, radius }` via `setTargeting()` (auto-pausing
   like a menu/overlay, see `setMenu`/`setOverlay` below) instead of
   applying the effect immediately; the next click, or ENTER on the
   keyboard cursor, calls `resolveTargeting(x,y)`, which only hits creeps
   (fireball/cold, via `creepCell`) or towers (charging, via `t.x,t.y`
   directly since towers don't move) within `radius` of that point, then
   consumes the inventory slot. `drawGame()` renders a live range-preview
   circle at the hover/cursor position while targeting is active (same
   `drawRange()` helper the menu's tower-range preview uses, different
   tint). Escape while targeting cancels without consuming the item, and
   all three hotkeys are only reachable from the base `keydown` switch
   (after the overlay-swallow and menu-key-match blocks), so a targeting
   arm can never fire while an overlay/menu is already open. Inventory in
   `G.inv`; view with the `i`
   overlay (`drawInventory`), which is directly usable rather than a pure
   readout — `invItems()` is the single source of truth for which items
   you're carrying and what each one's hotkey does; `invLayout()` adds
   the exact screen row each item's two lines land on (mirroring the
   tile-action menu's `menuLayout`/`menuBox` convention: `drawInventory()`
   and `mousedown`'s hit-test both derive their row math from the same
   function, so they can't disagree about where a row actually is).
   Pressing an item's hotkey OR clicking its row while `G.overlay ===
   "inv"` runs that item's action immediately and closes the overlay —
   handled by a dedicated block in `keydown` (before the generic
   overlay-swallow) and in `mousedown` (before the generic
   click-closes-overlay fallback in `toggleMenu`), since the whole point
   of opening the inventory is usually to decide what to use, not to
   memorize a hotkey and close it first. Every current consumable is
   aimed (`beginScroll`/`beginWand`/`beginCharge`'s shared pattern) —
   there is deliberately no blanket, map-wide item effect. If you add a
   new consumable, default to giving it a `beginX()`/targeting step too;
   only skip it with a specific reason (documented at the call site) if
   the effect genuinely can't be localized to a point. All game
   events go through `message(s)`, which both sets the message line and
   appends to `G.log` (viewed via the `log` overlay, `Ctrl-P`,
   NetHack-style). There is no floating combat text — kills, drops, leaks
   etc. are all reported as messages, not `G.fx`-style popups. A message
   ending in the literal string `"--More--"` renders that suffix in
   reverse video (real NetHack's own wording and look) — used for
   anything the player must consciously dismiss with SPACE (wave-clear,
   boss kill) rather than a message that just scrolls by.

   **Message variety:** the highest-repeat, narratively-flavored messages
   (kill, boss kill, leak, build, upgrade, sell, item drops, tower
   destroyed/disarmed, wave-clear, descend, haste potion) are each 2–4
   variants picked with `pick(arr)` (`arr[ri(0, arr.length-1)]`, defined
   next to `rnd`/`ri`) rather than a single fixed string, so a long
   session doesn't read the exact same sentence dozens of times. `pick()`
   draws from the same seeded `rnd()` as everything else, so a replayed
   `?seed=N` game shows the same sequence of message variants too, not
   just the same map/waves. Purely functional status/error messages
   (afford checks, "already fully enchanted," pause/speed/auto-start
   toggles, the one-time title/help text) were deliberately left as
   single strings — they're transient UI feedback, not narrative flavor,
   and don't benefit from variety the way a message you'll see 50+ times
   a game does. If you add a new frequently-repeating flavor message,
   default to giving it 2–3 variants via `pick()` rather than one fixed
   string, consistent with the existing ones.

   **Elemental damage & dragon immunity:** `damageCreep(c, dmg, dtype)`'s
   third argument is one of `"fire"|"acid"|"electric"|"cold"` (or omitted
   for untyped/physical, which always lands); if `dtype` matches the
   target's `MONSTERS[c.type].immuneTo`, the hit is fully absorbed (0
   damage, but `hitFlash` still fires so it doesn't look like nothing
   happened). Fire trap→fire, acid blob→acid, wand of lightning→electric,
   frost fountain→cold, wand of striking→untyped (always effective, the
   "reliable" tower), fireball scroll→fire. The demon is permanently
   fire-immune and the eel electric-immune; the dragon's color (and thus
   its immunity) is randomized per level by `pickDragon()` — called from
   `genWaves(dlvl)` right where wave 10's dragon group is added, picking
   one of `DRAGON_COLORS` with the seeded `rnd()`/`ri()` (so it's
   reproducible per dungeon like everything else). `pickDragon()`
   mutates the shared `MONSTERS.D` entry directly rather than threading a
   per-creep override through every render/message site — safe because
   there's only ever one dragon alive at a time (the lone wave-10 boss)
   and levels are strictly sequential, never concurrent. If you add a
   damage source, decide deliberately whether it has an elemental type
   (and pass it) or is physical (and doesn't) — the point of dragon
   immunity is that no single tower trivializes every level's boss, so a
   new AoE/instant-hit effect that forgets to type itself defeats that.
   A hit fully absorbed by immunity also shows a message the *first*
   time it happens to a given creep (`c.immuneMsgShown`, keyed per-creep
   not per-monster-type, so a fresh dragon on the next level gets its own
   first-hit message again) — flavor text keyed by `dtype` in
   `IMMUNE_FLAVOR` (e.g. "The fire bounces off its thick scales!"), not
   by monster, since it's the damage type failing, not the monster doing
   anything special. Only the first hit messages; every hit after that
   stays silent so a tower firing every `cd` tick doesn't spam the
   message line.

   **Siege monsters and trap disarming** are the counters to a "pipe of
   death" — a maze corridor lined solid with towers that, once built, is
   otherwise permanently safe. `siege: true` (orc captain, troll/
   chieftain, ogre, horned devil) lets a monster chip `SIEGE_DPS` off any
   orthogonally-adjacent *non-wall* blocking tower's `siegeHp` — a
   separate durability pool from the wall's own `hp`, since walls are
   already choppable by any creep and are meant to be the cheap
   sacrificial option; other towers needed their own field so a plain
   rat can't accidentally start whittling down a Wand of Lightning.
   `disarms: true` (gnome, dwarf) lets a monster standing directly on a
   fire trap (contact, not adjacency — traps don't block, so a monster
   can be on the exact same cell) wear down its `trapHp` at
   `DISARM_DPS` instead of triggering it. Both mechanics live in one
   function, `updateSiege(dt)` (called from `update()` right after
   `updateWalls(dt)`), dispatching on which field (`spec.siegeHp` vs
   `spec.trapHp`) a tower has; reaching 0 destroys the tower with no
   refund, same as a wall crumbling — siege recomputes `G.flow` since the
   tile becomes walkable again, disarming doesn't since traps never
   blocked movement. `buildTower`/`upgradeTower`/`resolveTargeting`'s
   `"charge"` branch (wand of charging) all treat `siegeHp`/`trapHp` exactly like `hp`/`maxCharges`
   already were — initialize on build, refill on enchant, eligible for a
   charging-wand dice roll — so a new destructible resource on a tower
   doesn't need a fourth copy of that logic, just one more `if`.

   **Monster lore popup:** clicking a live creep (when the tile has no
   tower — a tower's own upgrade/sell menu takes priority, e.g. a flying
   creep sharing a tower's cell) opens `G.lore` (a monster type key) via
   `setLore()`, which auto-pauses like an overlay. `drawLore()` renders
   the monster's `MONSTERS[type].lore` flavor text (word-wrapped by
   `wrapText()`) plus a stat/flag line (hp/spd/bounty and any of
   flies/submerges/splits/siege/disarms/regen/immuneTo it has). It's
   treated as a modal like the three big overlays (Escape or SPACE closes
   it and swallows every other key) rather than a lightweight popup like
   the tile-action menu, since there's nothing else useful to do while
   it's open. The dragon's `lore` field is mutated by `pickDragon()`
   alongside `name`/`fg`/`immuneTo` (see `DRAGON_COLORS`, each color
   entry carries its own `lore` string) so the popup always describes
   *this* level's dragon color, not a stale one from a previous descent.

   **Player level gates tower enchantment**, independent of gold:
   `G.plevel` (1–4) is computed from cumulative kill XP
   (`PLAYER_XP_THRESHOLDS`, `playerLevelForXp()`) via `gainXp()`, called
   from `damageCreep()`'s kill branch with `m.bounty` as the XP amount —
   deliberately called *last* in that branch (after the kill/boss
   message) so a level-up's own `--More--` message wins over the kill
   message rather than being immediately overwritten by it.
   `playerMaxTowerLvl()` (`Math.min(MAX_LVL, G.plevel)`) is the real cap
   `upgradeTower()` enforces; `MAX_LVL` itself never changes and is still
   the absolute ceiling once `G.plevel` catches up to it. A player level
   1 game can't enchant ANY tower past L1 even sitting on a full
   treasury — gold buys towers, XP (from actually fighting) buys the
   right to improve them. Like `G.gold`/`G.score`, `G.xp`/`G.plevel`
   persist across `descend()` and only reset in `resetGame()`.

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

   **Boss-kill pause:** killing a boss (`m.boss`) sets `G.paused = true`
   directly (not via `autoPause()` — see section 7) alongside its
   `--More--` message, so the player has to consciously acknowledge the
   kill with SPACE before anything continues. This is a genuine mid-frame
   hazard: `damageCreep()` is called from deep inside `updateTowers()`,
   and `update()` keeps running the rest of its body (`updateCreeps`,
   `updateWalls`, the wave-clear check) in that same frame regardless —
   so the wave-clear check is explicitly guarded with `!G.paused` to stop
   it from resolving `descend()`/`won` before the player ever sees the
   pause. If you add another mid-`update()` pause trigger, it needs the
   same guard or it'll be invisible one frame out of every game.
   `gainXp()`'s player-level-up message (see section 3) uses this exact
   same direct-`G.paused = true` + `--More--` pattern for the same
   reason — it's also reached from inside `damageCreep()`, deep in
   `updateTowers()`.

   **Pathfinding (maze-building TD, not a fixed-path TD):** most towers
   (`TOWERS[n].blocks === true` — everything except the fire trap)
   physically occupy their tile; monsters
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
   code path. Since a flyer's `c.x,c.y` genuinely CAN equal a tower's
   cell (that's the point — it flew over it), the creep-render loop in
   `drawGame()` tints the background (`#241a30`) whenever
   `m.flies && towerAt(x,y)` so it reads as "flying over" rather than
   looking like a rendering glitch.

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
   cell is room floor `.` AND the offset cell isn't occupied by a tower
   (`!towerAt(vx,vy)` — a real bug fixed here: the jitter used to check
   only terrain, so a ground creep whose true position was a legitimate
   tower-free cell could visually render, and be targeted/collide, on an
   adjacent tower's cell purely as a rendering-offset artifact; it never
   affected leak detection since that reads `c.x,c.y` directly, not
   `creepCell()`) — never in 1-wide corridors, so trap alignment and
   targeting stay exact regardless; it's used for everything: rendering,
   tower targeting, trap collision, and leak/hover checks. `creepProgress(c)`
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
   now. A creep that dies or leaks is removed from `G.creeps` in the very
   same `update()` call that killed/leaked it (`updateCreeps` filters
   dead creeps immediately, and `updateTowers` — which calls
   `damageCreep()` — runs earlier in the same `update()`), so without a
   marker it would never actually be rendered at all: gone between one
   `draw()` and the next with nothing to show it was ever there. A
   0.35s-`ttl` `G.fx` entry at the creep's cell (a `%` corpse glyph for a
   kill in `damageCreep()`'s kill branch, the creep's own glyph in red for
   a leak in `updateCreeps`' exit-cell branch) guarantees at least one
   visible frame regardless of exactly when in the update the death
   landed — most noticeable right at the stairs, since that's exactly
   where the player is watching closely for a leak.

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

7. **Input — tile-action menu, not a readied-tool model.** Clicking a
   tile (or moving the keyboard cursor and pressing `ENTER`) opens a
   small popup — `G.menu = { x, y }` — listing every action legal on
   *that specific tile*: an existing tower's `[u]pgrade`/`[s]ell`, or one
   `[1]`–`[6]` build option per tower type that's actually placeable
   there. `menuOptions(x, y)` computes this **fresh on every call**
   (never cached on `G.menu` itself) so it's always a live view of what's
   currently legal — e.g. re-opens automatically showing build options
   the instant you sell a tower, since the tile's options changed but
   `G.menu` still points at the same `{x,y}`. `menuLayout(x, y)` wraps
   it and adds `headerLines` (1 for an existing tower's live stats
   readout — dmg/range/charges or hp — 0 for a build menu, which has no
   single tower's stats to show); `menuBox(x,y,n)` computes the popup's
   on-screen box (clamped to stay within map rows/columns) from a total
   line count that must include `headerLines`. `drawMenu()` and the
   `mousedown` handler's click-hit-test both call `menuLayout`/`menuBox`
   the same way and derive each option's row from the same
   `by0 + 2 + headerLines` offset, so rendering and hit-testing can never
   disagree about where an option line actually is — if you change one
   side's row math you must change the other's identically. `toggleMenu(x,y)`
   opens/closes the menu (same tile twice = close). Executing an option
   normally closes the menu, EXCEPT upgrade/sell (`keepOpen: true` on
   those two), so you can keep enchanting the same tower, or immediately
   see rebuild options after selling, without reopening it. The old
   "press 1–4 to ready a tool, then click" model and its bottom-line tool
   list are gone — don't reintroduce a persistent tool-select control,
   that's the exact UX this replaced.

   The menu is a lightweight popup, not a modal: unmatched keys fall
   through to the rest of `keydown` while it's open (so `hjkl` cursor
   movement, item hotkeys, etc. keep working), unlike the three big
   overlays (`help`/`inv`/`log`, toggled by `?`/`i`/`Ctrl-P`) and the
   monster lore popup (`G.lore`, see section 3), which genuinely swallow
   everything else and are checked earlier in `keydown` for that reason.
   `Escape` OR `SPACE` closes whichever overlay/lore popup is open (real
   NetHack itself uses SPACE to dismiss `--More--`, so it reads naturally
   here); `Escape` alone closes the menu (SPACE keeps its other meanings
   there — see below — since the menu is a lightweight popup, not a
   modal). A click always tries the overlay/lore popup first (closes it
   without also opening a menu or triggering whatever's under the
   cursor) — a second click is needed to actually open a menu.

   **Opening an overlay, the menu, a targeting arm, or the lore popup
   auto-pauses a running wave**, and closing/resolving/cancelling it
   auto-resumes — `setOverlay(name)`/`setMenu(m)`/`setTargeting(t)`/
   `setLore(type)` are the only correct way to mutate
   `G.overlay`/`G.menu`/`G.targeting`/`G.lore`; they call
   `autoPause()`/`autoResume()` on the open/close transition. Never
   assign any of those four fields directly.
   `G.autoPaused` tracks *why* it's paused so a manual `SPACE` pause (or
   a boss-kill or player-level-up pause, see section 4 / section 3) is
   never silently undone by closing a menu/targeting arm/lore popup that
   happened to also be open — `autoResume()` only un-pauses if
   `G.autoPaused` is true, i.e. WE were the one who paused it.

   Bindings mirror NetHack where one exists (`r`ead, `q`uaff, `z`ap,
   `i`nventory, `Ctrl-P` message log); `h j k l y u2 b n` are cursor
   movement (note the fake `u2` key name — plain `u`/`s` are reserved for
   the menu's upgrade/sell, not diagonal cursor movement, so avoid
   reusing those two letters for anything else). `SPACE` is contextual:
   `startWave()` while `G.state === "build"`, otherwise toggles
   `G.paused` — except it's a no-op while `(G.menu || G.targeting ||
   G.lore) && G.autoPaused` is true, since pause state is governed by
   the open menu/targeting arm/lore popup in that case, not by SPACE.

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
  glyphs with classic meanings, but a glyph is no longer 1:1 with a
  monster identity — remember `MONSTERS` is now keyed by an id string
  with a separate `ch` (see section 3), so a new monster can either add
  a new tier to an existing glyph's family (a real-NetHack-authentic
  move — `d` canines, `o` orcs, `T` trolls, `&` demons all already do
  this) or claim an unused glyph outright.
- Colors: phosphor-green baseline (`#3d6b3d` floors, `#50ff96` accents);
  monsters/towers get muted CRT-friendly colors, full white reserved for
  hit-flash.
- No external assets: no CDNs, fonts, images, or fetches. The strict
  one-file constraint is the point of the project.
- Git: work on `main` and push there (user's standing instruction).
