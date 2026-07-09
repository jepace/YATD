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
3); idea 7 below has been updated to match. New blocking towers should
set `blocks: true` in `TOWERS` and will automatically participate in the
maze system; non-blocking (contact/aura) towers should set
`blocks: false` like the fire trap.

**Process note:** the user has asked that changes here be verified by
code review, not by launching a headless browser to click through the
game — see `CLAUDE.md`'s Testing section for the default (syntax-check
only) vs. when full browser verification is still warranted.

## High value, moderate effort

1. **Multiple dungeon levels (Dlvl 2+).** Refactor the map carving into
   `buildLevel(rooms, waypoints)`, add 2–4 more level definitions with
   distinct topologies (a big open cavern, a spiral, a double-path split).
   Beating wave 10 descends to the next level, carrying gold and score;
   status bar already shows `Dlvl:`. Difficulty rescales per level.
2. **High-score "bones file."** Persist best scores in `localStorage`
   with NetHack-style entries ("Keeper the Conjurer, slain on Dlvl 2 by a
   zombie horde"). Show top 5 on the title and death screens.
3. **Sound via WebAudio.** No assets — synthesize terminal-appropriate
   bleeps/square waves: build chirp, zap, monster death, leak thud, boss
   roar, victory arpeggio. `m` to mute, default ON but quiet.
4. **More monster behaviors.** Cheap wins with the existing creep struct:
   `;` giant eel (submerges: untargetable every few seconds), `&` demon
   (gates in a copy at half HP on death), `L` lich (casts haste on nearby
   monsters), `x` grid bug (cannot move diagonally — flavor text only,
   they already can't). Add waves 11–15 or weave into higher Dlvls.

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
7. **Tower variety.** `\` Lightning rod (chain to 3 targets, needs LOS
   like the other ranged towers), `)` Hired adventurer (blocks the
   corridor, melees, has HP, monsters attack it — no LOS needed, it's
   melee), `"` Amulet shrine (aura buff to adjacent towers). Keep total
   ≤ 7 with hotkeys 1–7; pick glyphs from unused NetHack object-class
   punctuation, never a letter (see the process note above).
8. **Endless mode.** After wave 10 victory, offer "descend anyway":
   procedurally scaled waves (reuse `waveHpScale`, cycle monster mixes,
   boss every 5) until death; score-chasing pairs well with idea 2.

## Polish

9. **Better projectile/impact feedback.** Traveling `*` projectiles for
   the acid lob (currently instant), screen-shake (1-cell jitter) on
   boss death and leaks, brief red flash of the status HP when hit.
10. **Intro/outro flavor.** Randomized NetHack-style level entry messages,
    an oracle room with hint text, `--More--` prompt for multi-line
    events.
11. **Mobile/touch support.** Tap = click already works; add a bottom
    touch toolbar for tower selection and wave start when
    `pointer: coarse`.
12. **Difficulty settings.** Title-screen choice: Tourist (easy) /
    Valkyrie (normal) / Wizard (hard) mapping to starting HP/gold and
    `waveHpScale` slope.

## Explicitly rejected (don't redo these debates)

- Frameworks, bundlers, npm deps, external fonts/CDNs — the single
  self-contained file is a core feature.
- DOM-per-cell rendering — canvas is faster and simpler.

(Maze-building pathfinding was previously on this list as rejected —
it's now shipped, see the top of this file. Towers block, monsters
route around them live.)
