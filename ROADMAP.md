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
multiple dungeon levels (`descend()` past each wave 10, now endless,
difficulty scaling with depth).

**Most recently shipped:**
- **No more level caps on either side.** Player level and tower enchant
  level are unlimited (`xpForLevel` continues the XP table at 2.3x per
  level; `MAX_LVL = Infinity`; tower range alone stops growing at L6).
  Past Dlvl 5, bounty/XP grow 1.1x/level, and elite "greater" monsters
  (1.5x HP, double bounty, one extra ability, read via `trait(c,k)`)
  appear at 6%/level up to 50% of spawns. The player's scaling is kept
  just behind the dungeon's on purpose so runs still end. See `CLAUDE.md`
  sections 3 and 4.
- **Endless dungeon, and a real tombstone.** The Dlvl 5 victory is gone:
  every wave-10 clear descends again, forever, until you die. Spawn
  counts grow exponentially only through Dlvl 5 then nearly flat, with
  deep-level difficulty carried by HP plus one random `LEVEL_MODS` twist
  per level from Dlvl 6. The tombstone is NetHack's own RIP art drawn as
  one aligned block (the old one centered each line separately, which is
  why its walls were ragged), naming what killed you in its own color,
  your gold, depth and the year, with an epitaph and — answering its own
  "possessions identified?" question — the consumables you died without
  using. See `CLAUDE.md` section 4.
- **Unaffordable menu options are greyed out.** Every cost-gated option
  in the tile-action menu (upgrade, unlock, build — not sell, which has
  no cost) now renders dim grey when you can't currently afford it or
  (for upgrade) don't have the player level for it, instead of looking
  identical to something you can actually do. Purely a rendering hint —
  the option is still clickable and still runs its normal action with
  its own affordability check, so there's only one source of truth for
  "can I afford this," reused for both the color and the click. See
  `CLAUDE.md` section 7.
- **Player level pacing steepened, and the cap raised from 4 to 6.**
  Follow-up to a report that player level (and thus the tower enchant
  ceiling) was still climbing too fast even after the previous economy
  pass. `PLAYER_XP_THRESHOLDS` went from `[0,100,300,800]` to
  `[0,150,500,1400,3200,7000]` — steeper (~2.5-3x per level instead of
  ~2.7x from a much lower base) and two levels longer. `MAX_LVL` (the
  absolute tower enchant ceiling, `playerMaxTowerLvl`'s cap) went from 4
  to 6 alongside it, so there's real new headroom above where the old
  cap sat, not just a slower climb to the same wall. No other code
  needed to change — `UPG_MULT`/charge/HP scaling and `upgradeCost` all
  already generalize to any `t.lvl`, they just hadn't had reason to go
  past 4 before.
- **Abandoned towers are auto-refunded on descend.** `descend()` used to
  clear `G.towers` with zero refund, and there's no real window to sell
  manually beforehand since you still need those towers to survive wave
  10. Now sums `Math.floor(t.spent * DESCEND_REFUND_MULT)` (0.4, worse
  than `sellTower`'s manual 0.7) across all towers before clearing them
  and folds it into the descend message. See `CLAUDE.md` section 4.
- **Clearer feedback when a build fails because a monster is mid-step
  onto that tile.** Creeps render only at their discrete cell, not
  interpolated, so a tile they're already committed to stepping into
  (`creepReserves(x,y)`) could look completely empty on screen while
  still correctly refusing a build. `buildTower()` now gives a distinct
  message for that case, and the hovered tile gets a subtle tint.
- **Room walls no longer get a stack of redundant doors.** A corridor
  that ran flush alongside a room's wall for a stretch used to turn
  every one of those cells into a door — one wide gap with several
  doorways stacked on it, not a real doorway. `genRooms()` now rejects
  (or, for the entrance/exit's forced rooms, re-rolls) any room
  placement whose wall would run flush alongside 2+ other path cells;
  verified with a 500-seed scratch test that this doesn't regress
  connectivity (see the corner-door regression above — this kind of
  change gets a same-shape connectivity check every time now).
- **Dirt Wall starts unlocked alongside the Archer.** `G.unlocked`'s
  fresh-game default is now `{ 1: true, 6: true }` instead of just
  `{ 1: true }` — a new player has a way to shape the maze from turn one
  without needing to save up unlock gold first. `TOWERS[6]`'s
  `unlockCost` field was removed since it's no longer meaningful.
- **Wand of charging is now targeted, not a blanket effect.** Follow-up
  to a report that it "just charges everything" while its two sibling
  consumables are aimed — `c` now arms `G.targeting` (`beginCharge()`,
  radius 3.0) exactly like the fireball scroll and cold wand, and only
  towers within that radius get the dice-roll restore. Reinforces the
  standing rule that nothing in this game is a map-wide effect; see
  `CLAUDE.md` section 3.
- **Message variety.** The highest-repeat flavor messages (kill, boss
  kill, leak, build, upgrade, sell, item drops, tower destroyed/disarmed,
  wave-clear, descend, haste potion) now each pick from 2–4 variants via
  a new `pick(arr)` helper (seeded off the same `rnd()`, so `?seed=N`
  replays are still deterministic) instead of one fixed string — a long
  session no longer reads the exact same sentence 50+ times. Purely
  functional status/error messages were deliberately left alone; see
  `CLAUDE.md` section 6 before adding a new one-off flavor message.
  Also fixed a real grammar bug this surfaced: the "You conjure a/an X"
  article was hardcoded to only special-case tower type 1, so "Acid
  Blob" (also vowel-initial) was getting "a Acid Blob" — now computed
  generically from the tower's name.
- **Two dungeon-generation correctness fixes, one of which shipped a
  regression that got caught and fixed in the same session.** Stairs
  (`<`/`>`) could previously land on a bare corridor cell or even a
  wall, since `ENTRANCE`/`EXIT` were just wherever the first/last path
  waypoint happened to fall; `genRooms()` now force-places a room on
  each one first, guaranteeing real room floor — this part is solid.
  Separately, a corridor that merely grazed a room's corner could get
  turned into a door there — diagonal-only from the room's actual floor,
  so it looked connected but wasn't reachable — and the first fix
  excluded *every* room corner from door conversion unconditionally.
  That broke roughly 2/3 of generated levels: a corridor can legitimately
  run straight through a corner as part of its own main route (not just
  graze it diagonally), and severing those disconnected the entrance
  from the exit entirely. Fixed properly: a corner is only skipped as a
  door site when NEITHER orthogonal neighbor is also on the path
  (`isolatedCorner()`) — a corner with a path neighbor on either side is
  load-bearing and still becomes a door. See `CLAUDE.md` section 2 for
  the full story and, more importantly, the regression test pattern to
  reuse if you ever touch this code again: score/path-length passing the
  existing quality gate does NOT imply the level is actually connected —
  you have to check `computeFlow().dist` at `ENTRANCE` is finite with
  zero towers built, across many seeds.
- **Tower unlocking, and an early-game economy tune-down.** A second
  progression axis alongside player level: only the Archer (`TOWERS[1]`,
  renamed from Wand of Striking) is available from a fresh game — every
  other tower type needs a one-time flat gold payment
  (`unlockTower(type)`, `TOWERS[n].unlockCost`) before it shows up as
  buildable, permanently, surviving `descend()` same as gold/player
  level. See `CLAUDE.md` section 3. Alongside it: `BOUNTY_MULT` (0.75)
  now trims gold income and XP together, and `PLAYER_XP_THRESHOLDS` was
  raised, after a report of reaching max player level and a huge gold
  surplus by wave 9 of Dlvl 1 with every tower still at L1 — the early
  economy was too generous to ever feel like resources mattered.
- **Siege monsters, trap disarming, player XP levels, and monster lore.**
  Four related additions in one pass:
  - Tower-smashing monsters (`siege: true` — orc captain, troll/troll
    chieftain, ogre, horned devil) chip `SIEGE_DPS` off any adjacent
    non-wall blocking tower's own `siegeHp` pool, the counter to a "pipe
    of death" corridor that's permanently safe once built. Trap-disarming
    monsters (`disarms: true` — gnome, dwarf) do the same to a fire
    trap's `trapHp` by standing on it instead of triggering it. Both live
    in `updateSiege(dt)` — see `CLAUDE.md` section 3.
  - Player level (`G.plevel`, shown as `Lv:N`) is now the real cap on
    tower enchantment, separate from gold — earned via kill XP
    (`gainXp()`, `PLAYER_XP_THRESHOLDS`), announced with its own
    pause-for-SPACE message like a boss kill. `MAX_LVL` is still the
    absolute ceiling; `playerMaxTowerLvl()` is `min(MAX_LVL, G.plevel)`.
  - Click any monster (`G.lore`, `setLore()`/`drawLore()`) to read its
    name, stats, special-ability flags, and a line of NetHack flavor
    text — the dragon's lore is color-specific and re-picked with the
    rest of `DRAGON_COLORS` each level.
  - A damage source that's fully absorbed by elemental immunity now
    messages the player the first time it happens per creep ("The fire
    bounces off its thick scales!") instead of silently doing nothing.
  - Also: `SPACE` now closes the inventory/log/help overlay (and the new
    lore popup), not just `Escape`; Wand of Striking's cost went from $10
    to $18 (it was priced under the $8 Dirt Wall despite being an actual
    attack tower, making it strong enough to solo a run).
- **~30-monster roster with per-dungeon-level gating.** `MONSTERS` grew
  from 11 entries to ~30, now keyed by an id string with its own `ch`
  (glyph), decoupled from the key — several tiers share a letter the way
  real NetHack's monster classes do (`d` canines jackal→wolf→warg, `o`
  orcs, `T` trolls, `&` demons). `genWaves`'s pool entries are now gated
  on `minDlvl`/`maxDlvl` in addition to the existing `min` (wave-within-
  level): grid bugs, for instance, are gone by Dlvl 3 instead of showing
  up at any depth just because it's wave 1 of that level. Special
  abilities (flying, submerging, splitting, fast regen) are spread
  across 2–4 monsters apiece now instead of one signature monster each,
  for more run-to-run variety. See `CLAUDE.md` section 3 before adding
  or re-tiering an entry — there's an invariant about keeping at least
  one uncapped `min: 1` pool entry so no (dlvl, wave) combo ever comes
  up with zero eligible monsters.
- **Gold value scales down with depth.** `towerCost(type)` multiplies
  every build/enchant price by `1.18^(dlvl-1)` on top of the base cost
  in `TOWERS`, so the gold you're still sitting on by Dlvl 5 doesn't go
  as far as it did on Dlvl 1 — addresses reports of finishing runs with
  thousands of unspent zorkmids. Difficulty scaling per dungeon level
  was also bumped a notch (`waveHpScale`'s dlvl exponent 1.15→1.2, the
  boss's matching exponent, and `genWaves`'s per-dlvl budget multiplier
  1.3→1.35) after a report of clearing all of Dlvl 5 without leaving the
  entrance room — see `CLAUDE.md` section 3 for exact formulas before
  retuning.
- **Targeted scroll/wand deploy.** The fireball scroll and cold wand no
  longer hit every monster on the level instantly — `r`/`z` arms a
  targeting reticle (`G.targeting`, auto-pausing like a menu), a click or
  `ENTER` on the cursor picks the impact point with a live range-preview
  circle, and only creeps within that radius are affected. See
  `CLAUDE.md` section 3 (`resolveTargeting`) and section 7
  (`setTargeting`) before touching it.
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
- **Wand of charging** (item, `c` to zap) — restores `t.lvl` dice of
  `d6` to every tower's charges/HP that's below max, deliberately not a
  guaranteed full top-off (real NetHack precedent, and the broader
  "nothing is perfect, everything is useful" design note above).
  Applies to any tower with `spec.maxCharges` or `spec.hp` set, no
  per-type special-casing, so wand of lightning benefits identically to
  wand of striking.
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
3. **Give the lich (`L`) its signature ability.** It's shipped as a
   plain high-HP/regen Dlvl-5 monster (see "most recently shipped"
   above) but not yet the actual support caster from the original idea:
   a haste aura on nearby monsters — proximity check among `G.creeps`
   each frame, cheap at this creep count.
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
6b. **Blessed/uncursed/cursed items** (not the same idea as 6 — this is
   real NetHack's per-item quality system, raised and deliberately
   deferred in an earlier session). The real cost isn't the effect
   table, it's that `G.inv` currently holds plain stack counts
   (`G.inv.scroll`, `G.inv.wand`, ...) with no per-item identity to hang
   a quality on — doing this properly means reworking inventory into
   arrays of `{quality}` entries, updating `drawInventory()` to show
   NetHack's `B`/`U`/`C` prefixes, and deciding a deliberate
   effect-magnitude swing per item type (what does a *cursed* scroll of
   fireball or wand of cold actually do — fizzle? backfire on the
   player's own towers? just weaker?) consistent with the "nothing is
   perfect" design note in `CLAUDE.md` section 3. Suggested odds if
   picked up: roughly NetHack's own ~1/6 blessed, ~1/6 cursed, rest
   uncursed. Get explicit sign-off on the odds and the cursed-effect
   design before starting — it touches the data model, not just a table.
7. **More tower variety.** `)` Hired adventurer (blocks the corridor,
   melees, has HP like the dirt wall but actually fights back, monsters
   attack it — no LOS needed, it's melee), `"` Amulet shrine (aura buff
   to adjacent towers, no attack of its own). We're at 6 of a soft cap of
   7 (hotkeys 1–9 would still fit on the status line if that cap needs to
   move); pick a glyph from unused NetHack object-class punctuation,
   never a letter (see the process note above).
8. ~~Endless mode~~ — shipped: the dungeon is now endless by default
   (see "most recently shipped"). Score-chasing makes idea 1 (bones /
   high-score file) more valuable than ever — `G.dlvl`, `G.killedBy` and
   `G.epitaph` are already tracked for it.

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
