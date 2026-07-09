# YATD Roadmap — enhancement ideas

Ideas from the original build session, roughly ordered by
bang-for-the-buck. Each should keep the one-file, zero-dependency,
NetHack-voiced constraints described in `CLAUDE.md`.

**Already shipped since v1:** pause (now `SPACE` while a wave is running;
starts the wave when in build phase), procedural dungeon generation
(seeded, `?seed=N`), randomized wave composition (`genWaves` budget
system), random item drops (scroll of fireball / potion of haste / wand
of cold / gold piles) with an `i` inventory overlay, a `Ctrl-P` message
log (NetHack-style — replaced the old floating "+2$" combat text
entirely; all events are `message()` calls now), non-sticky tower
placement (`1`-`4` must be pressed again before each build), creep lane
variation in rooms (`jit`), line-of-sight for ranged towers
(`hasLineOfSight`, walls block shots — traps are exempt), and **live
maze-building pathfinding**: towers 1-3 physically block their tile and
monsters route around them via a BFS flow field recomputed on every
build/sell (`computeFlow`/`G.flow`), with `wouldSealPath` refusing any
placement that would wall the stairs shut entirely. See `CLAUDE.md`
section 4 before touching monster movement or tower placement rules —
it's easy to introduce a softlock (a wave that can never finish because
a creep has no route) if you change the connectivity check without
understanding it first. New item ideas should extend `dropItem()` and
the `G.inv` + `r/q/z` key pattern. New overlays should follow the
`G.overlay` + `overlayBox()` pattern in
`drawHelp`/`drawInventory`/`drawLog` — see `CLAUDE.md` section 6. New
towers must use NetHack object-class punctuation for their glyph, never
a letter — letters are exclusively monsters now (see `CLAUDE.md` section
3); idea below has been updated to match. New blocking towers should
set `blocks: true` in `TOWERS` and will automatically participate in the
maze system; non-blocking (contact/aura) towers should set
`blocks: false` like the fire trap. Also shipped: **limited wand
charges** (`TOWERS[1].maxCharges`, regenerates passively, refills on
enchant — the counter to a wall of cheap wands needing zero upkeep),
**four new monster behaviors** — `x` grid bug (plain filler, weak/fast/
cheap), `b` cave bat (`flies: true`, ignores every blocking tower via a
separate `G.flowFly` field, the counter to turtling behind a maze), `;`
giant eel (`submerges: true`, periodically untargetable — see
`isSubmerged()` in `CLAUDE.md` section 3), `&` demon (`splits: true`,
spawns a weaker copy on death via `spawnChild()`) — **a fourth upgrade
tier** (`MAX_LVL` raised 3→4, with a matching 4th tower-color tier and
charge-scaling that already extended cleanly since the formulas were
level-generic), and **multiple dungeon levels**: clearing wave 10 below
`MAX_DLVL` (5) calls `descend()` — a brand new `genLevel()`/`genWaves()`,
gold/score/HP carry over, towers don't, difficulty scales up ~1.3x
budget and ~1.15x HP per level (bosses too). See `CLAUDE.md` section 4
("Dungeon levels") before touching level-transition logic. Also shipped:
**two more towers**, `\` Wand of Lightning (`kind: "chain"`, arcs to 2
more targets, `chainRange`/`chainFalloff` — this used to be idea 7's
"Lightning rod" suggestion below, now removed from the open list) and
`%` Dirt Wall (`kind: "wall"`, pure obstacle with `hp` instead of
`dmg`/`range`, worn down by `updateWalls()` whenever a live creep is
orthogonally adjacent — see `CLAUDE.md` section 3); **a real bug fix**
in `creepCell()`'s lane-jitter offset, which used to check only terrain
and could visually place a grounded creep on top of an adjacent tower
(never affected leak detection, which reads `c.x,c.y` directly, but did
affect what got rendered/targeted/collided) — now also checks
`!towerAt(vx,vy)`; and a **render fix** for flying creeps, which
legitimately CAN share a cell with a tower (that's the point of flying)
— the creep-render loop now tints the background so it reads as "flying
over" instead of looking like the same bug.

**Process note:** the user has asked that changes here be verified by
code review, not by launching a headless browser to click through the
game — see `CLAUDE.md`'s Testing section for the default (syntax-check
only) vs. when full browser verification is still warranted.

## High value, moderate effort

1. **High-score "bones file."** Persist best scores in `localStorage`
   with NetHack-style entries ("Keeper the Conjurer, slain on Dlvl 2 by a
   zombie horde" — `G.dlvl` is now tracked, use it). Show top 5 on the
   title and death screens.
2. **Sound via WebAudio.** No assets — synthesize terminal-appropriate
   bleeps/square waves: build chirp, zap, monster death, leak thud, boss
   roar, victory arpeggio, a distinct descend chime. `m` to mute, default
   ON but quiet.
3. **Lich (`L`).** The one remaining monster idea from the original list:
   casts haste on nearby monsters — proximity check among `G.creeps` each
   frame, cheap at this creep count. Gate it to a mid/high `min` wave or
   a minimum `dlvl` in the `genWaves` pool (support casters should show
   up once the player has real infrastructure to lose).
4. **Level topology variety.** Right now every Dlvl uses the same
   `genPath`/`genRooms` shape, just re-rolled and scaled harder. Add 1-2
   alternate topology generators (a big open cavern, a branching Y-split)
   and pick one per descent — `buildLevel()` already isolates the carving
   logic, so this is a new `genPath`-like function plus a switch in
   `buildLevel()`, not a rewrite.

## Medium

5. **Wand of Digging (consumable).** Now that movement is a live flow
   field, this is much easier than it used to be: flip a rock `' '` cell
   to floor `'.'` (or vice versa, collapsing a corridor into rock — check
   `wouldSealPath`-style reachability first so you can't softlock the
   level) and call `G.flow = computeFlow()`. Monsters crossing that area
   will pick up the new route on their next `advanceTarget()` call
   automatically, no other changes needed.
6. **Cursed/blessed waves.** Random modifiers announced in advance ("You
   feel a malignant aura..."): monsters hasted, towers -25% range, double
   bounty, fog (map dimmed outside tower ranges). One modifier every few
   waves keeps builds honest.
7. **More tower variety.** `)` Hired adventurer (blocks the corridor,
   melees, has HP like the dirt wall but actually fights back, monsters
   attack it — no LOS needed, it's melee), `"` Amulet shrine (aura buff
   to adjacent towers, no attack of its own). We're at 6 of a soft cap of
   7 (hotkeys 1–9 would still fit on the status line if that cap needs to
   move); pick a glyph from unused NetHack object-class punctuation,
   never a letter (see the process note above).
8. **Endless mode past Dlvl 5.** After the `MAX_DLVL` victory, offer
   "descend anyway": keep calling `descend()` past the current cap
   (`won` no longer ends things, or offer it as a post-victory choice)
   with waves continuing to scale via the existing `dlvlMul`/HP-scale
   formulas until death; score-chasing pairs well with idea 1.

## Polish

9. **Better projectile/impact feedback.** Traveling `*` projectiles for
   the acid lob (currently instant), screen-shake (1-cell jitter) on
   boss death and leaks, brief red flash of the status HP when hit.
10. **Intro/outro flavor.** Randomized NetHack-style level entry messages
    per Dlvl (beyond the current single descend message), an oracle room
    with hint text, `--More--` prompt for multi-line events.
11. **Mobile/touch support.** Tap = click already works; add a bottom
    touch toolbar for tower selection and wave start when
    `pointer: coarse`.
12. **Difficulty settings.** Title-screen choice: Tourist (easy) /
    Valkyrie (normal) / Wizard (hard) mapping to starting HP/gold and
    `waveHpScale`/`dlvlMul` slope.

## Explicitly rejected (don't redo these debates)

- Frameworks, bundlers, npm deps, external fonts/CDNs — the single
  self-contained file is a core feature.
- DOM-per-cell rendering — canvas is faster and simpler.

(Maze-building pathfinding was previously on this list as rejected —
it's now shipped, see the top of this file. Towers block, monsters
route around them live.)
