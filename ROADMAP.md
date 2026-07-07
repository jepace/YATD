# YATD Roadmap — enhancement ideas

Ideas from the original build session, roughly ordered by
bang-for-the-buck. Each should keep the one-file, zero-dependency,
NetHack-voiced constraints described in `CLAUDE.md`.

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

5. **Wand of Digging (consumable).** Player buys a one-shot item that
   carves a shortcut corridor between two chosen rock cells... for the
   MONSTERS' benefit? No — inverted: it collapses a corridor segment,
   forcing a longer path. Requires making `path` rebuildable (BFS over
   corridor cells instead of the fixed waypoint walk).
6. **Cursed/blessed waves.** Random modifiers announced in advance ("You
   feel a malignant aura..."): monsters hasted, towers -25% range, double
   bounty, fog (map dimmed outside tower ranges). One modifier every few
   waves keeps builds honest.
7. **Tower variety.** `L` Lightning rod (chain to 3 targets), `@` Hired
   adventurer (blocks the corridor, melees, has HP, monsters attack it),
   `"` Amulet shrine (aura buff to adjacent towers). Keep total ≤ 7 with
   hotkeys 1–7.
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
- Free creep pathfinding around towers ("maze-building" TD) — YATD is a
  fixed-path TD by design; towers go in rooms, traps on corridors.
- DOM-per-cell rendering — canvas is faster and simpler.
