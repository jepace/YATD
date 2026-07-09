# YATD Roadmap — enhancement ideas

Ideas from the original build session, roughly ordered by
bang-for-the-buck. Each should keep the one-file, zero-dependency,
NetHack-voiced constraints described in `CLAUDE.md`.

**Already shipped since v1** (deep mechanics for all of this are in
`CLAUDE.md`, not repeated here): procedural dungeon generation (seeded,
`?seed=N`), randomized wave composition, random item drops with an `i`
inventory overlay, a `Ctrl-P` message log (all events are `message()`
calls, no floating combat text), live maze-building pathfinding (towers
physically block, monsters route around via a BFS flow field,
`wouldSealPath` refuses any placement that would seal the stairs shut
entirely), 6 tower types incl. limited wand charges (now on both wands,
not just the striking one) and a destructible dirt wall, a 4th upgrade
tier, 6 monster types beyond the original 5 (flying bat, submerging eel,
splitting demon, grid bug filler) plus universal passive regen, and
multiple dungeon levels (`descend()` past each wave 10, up to `MAX_DLVL`
5, difficulty scaling with depth).

**Most recently shipped:**
- **Tile-action menu replacing the readied-tool model.** Click a tile
  (or `ENTER` the cursor onto it) for a popup listing every action legal
  there — build options, or an existing tower's upgrade/sell. No more
  "press 1-6 to ready a tool, then click." See `CLAUDE.md` section 7
  before touching `menuOptions`/`menuLayout`/`menuBox` — rendering and
  click-hit-testing derive their row math from the same functions on
  purpose, so they can't disagree about where an option line is.
- **Auto-pause/resume on any overlay or menu.** `setOverlay`/`setMenu`
  are the only correct way to open/close one — see `CLAUDE.md` section 7
  for why (`G.autoPaused` distinguishes an auto-pause from a manual one
  so closing a menu never steals a `SPACE` pause or a boss-kill pause).
- **NetHack's real `--More--` convention** for anything the player must
  consciously dismiss (wave-clear, boss kill) — renders in reverse
  video, not just appended text.
- **Boss-kill pause**, `SPACE` to continue — see `CLAUDE.md` section 4
  for the mid-`update()` hazard this had to guard against.
- **Elemental damage & dragon immunity.** `damageCreep(c, dmg, dtype)`;
  fire trap/acid blob/wand of lightning/frost fountain/fireball scroll
  are typed, wand of striking stays untyped (always effective — that's
  its whole point now). Demon is always fire-immune, eel always
  electric-immune. Each level's dragon is a random color with a
  matching immunity (`pickDragon()`, `DRAGON_COLORS`), so one tower type
  can no longer trivialize every boss fight.
- **Wand of charging** (item, `c` to zap) — instantly maxes every
  tower's charges/HP at once.
- A real **jitter/tower-occupancy bug fix** (a grounded creep's
  cosmetic lane-wobble could render it on top of an adjacent tower) and
  a **render fix** for flying creeps (which legitimately CAN share a
  cell with a tower — that's the point of flying — now tinted so it
  reads as "flying over" instead of looking like the same bug).

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
    with hint text. (The `--More--` convention itself is shipped — reuse
    it for these too, don't invent a second "press a key" pattern.)
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
