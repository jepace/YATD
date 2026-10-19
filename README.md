# YATD — Yet Another Tower Defense

A NetHack-flavored, ASCII tower defense game that runs entirely in your
browser and looks like it's playing on a green-phosphor CRT terminal.

**Play it:** just open `index.html` in any modern browser. No build step,
no dependencies, no network — the whole game is one file.

```
python3 -m http.server   # or double-click index.html
```

## The pitch

Monsters pour out of the up staircase `<` and find their own way to the
down staircase `>` — there's no fixed route, they route live around
whatever you build. Every monster that descends costs you HP. Conjure
towers to block and defend the dungeon, enchant them, and survive all 10
waves — the last one brings a red dragon.

Clear wave 10 and you descend to the next dungeon level instead of the
game ending: a brand new map, a harder wave set, and the same gold and
HP you finished with (plus a small heal for the stairs down). **The
dungeon has no bottom** — it keeps going, and getting harder, until it
kills you; your score and how deep you got are what you're chasing. From
Dlvl 6 on, every level also rolls a random twist, announced as you
arrive: swarms of weaker monsters, fewer but far tougher brutes, fleet
monsters that rush the stairs, a gold-rich hoard level, and so on. Towers don't carry over between
levels, so every floor is a fresh maze to build — but they're not a
total loss either: everything left standing when you descend gets
auto-refunded at 40% of what you paid for it (worse than the 70% you'd
get selling by hand, so sell first if you can, but you're never just
throwing a full build away).

Like NetHack, **every game is different**: every dungeon level is
freshly procedurally generated, wave compositions are randomized, and
monsters randomly drop items. Add `?seed=N` to the URL to replay a
particular first dungeon (the seed is shown on the title screen).

Some levels fork: a second corridor gives monsters two routes to the
stairs, so you have to maze both (or wall one off and defend the other).

## Items

Slain monsters sometimes drop loot: piles of zorkmids, `?` scrolls of
fireball, `!` potions of haste (`q`uaff to double your towers' speed for
8s), `/` wands of cold, and `/` wands of charging (restores charges/HP
to towers near where you aim it — like the real NetHack item, it's a
dice roll, not a guaranteed top-off, and rolls better on your
higher-level towers). Press `i` to check your pack.

The scroll of fireball, wand of cold, and wand of charging are all
**targeted**: `r`ead or `z`ap/`c`harge arms it, then click a tile (or
move the keyboard cursor there and press `ENTER`) to choose the impact
point — a preview circle shows the radius as you aim. Only monsters (or,
for the charging wand, towers) within that radius are affected;
`Escape` cancels without consuming the item. Fireball burns everything
in a 4-tile radius for 45 fire damage; the wand of cold slows everything
in a 4.5-tile radius for 4s; the wand of charging restores every tower
within a 3-tile radius. Nothing in this game is a blanket, map-wide
effect — you always have to commit to a spot.

Rarely a level has a shopkeeper (`@`). Click them to buy a few
consumables with gold. Stock is small and random, and it doesn't restock.

## Towers

Tower glyphs are always NetHack object-class punctuation, never a
letter — letters are reserved for monsters, so nothing on the map is
ever ambiguous about what it is.

| Key | Glyph | Tower             | Cost | Unlock | What it does                    |
|-----|-------|-------------------|------|--------|----------------------------------|
| `1` | `/`   | Archer            | $18  | free   | Fast single-target bolts, limited arrows |
| `2` | `0`   | Acid Blob         | $25  | $40    | Lobbed splash damage             |
| `3` | `{`   | Frost Fountain    | $20  | $35    | Chills and slows monsters in range |
| `4` | `^`   | Fire Trap         | $15  | $30    | Burns monsters — place ON the corridor |
| `5` | `\`   | Wand of Lightning | $30  | $70    | Arcs to 2 more targets after the first |
| `6` | `%`   | Dirt Wall         | $8   | free   | Pure obstacle, no offense — see below |

Prices above are Dlvl 1 baseline — every build and enchant costs more
the deeper you descend (gold you're still holding onto by Dlvl 5 buys
noticeably less than it did up top), so don't bank on stockpiling early
and coasting on it later.

**Only the Archer and Dirt Wall are available from a fresh game** — the
cheap, no-refund maze-shaping basics. Every other tower type has to be
unlocked first with a one-time gold payment — the
tile-action menu shows `Unlock <tower> ($cost, one-time)` in place of
the normal build option on any tile where that tower could go, and
paying it adds the type to your toolkit for the rest of the run (it
never re-locks, and survives descending to the next dungeon level —
only a full restart resets it). Unlock cost is separate from the
per-build price above and doesn't scale with dungeon level. This is a
second axis of progression alongside player level: player level (see
Monsters below) gates how far a tower you already have access to can be
enchanted; unlocking gates whether you have access to that tower type at
all. Which type you unlock first, and when, is a real strategic choice.

Both wands run on charges — they fire until dry, then sit idle (dimmed
on the map) recharging slowly on their own. Enchanting one tops it back
up and raises its max charges. Nothing in this game is a one-way meter:
wands drain and recharge, walls take damage and can be repaired,
monsters take damage and slowly heal. Don't rely on a wall of wands
alone to carry a wave — sustained damage output has a ceiling.

Every tower except the fire trap needs a clear line of sight — thick
walls block shots, so a tower can't hit monsters marching down the
corridor on the other side of a wall. The Wand of Lightning's arcs each
need their own line of sight too, checked from wherever the bolt just
hit, not from the tower.

The Dirt Wall has no attack at all — it's the cheapest way to shape a
maze, but it has HP, and any monster standing next to it chips away at
it over time. Leave one somewhere out-of-the-way and it'll last; put one
across a heavily-trafficked corridor and expect to rebuild it. Enchanting
one reinforces and fully repairs it.

All towers except the fire trap and dirt wall physically occupy their
tile — build one in a monster's way and it routes around, same as any
wall (the dirt wall blocks too, it just can't fight back). Build a maze
if you want, but you can never wall the stairs shut completely: the game
refuses any placement that would leave no route at all from `<` to `>`.
The fire trap is the odd one out — it lies flush with the floor, doesn't
block anything, and only goes directly on a corridor or doorway tile
where something will actually walk over it.

Every tower except the fire trap can also be smashed by the right kind
of monster — a corridor lined solid with towers isn't automatically
permanent. A handful of brutish monsters (see below) will chip away at
any tower standing next to them, same as they would a dirt wall, until
it's destroyed with no refund. And the fire trap, powerful as it is, has
its own weakness: a couple of monster types know how to disarm one by
simply standing on it.

Towers can be enchanted (`u`) for more damage, range (up to L6),
(wands) charges, or (the wall) HP, or sold (`s`) for a 70% refund
(destroyed walls give nothing back, since there's nothing left to sell).
Enchanting is also capped by your **player level** (see Monsters below,
"gaining experience") — a fresh player level 1 can't push any tower past
L1 even with gold to spare.

**Altars.** About one dungeon level in five has an altar (`_`) in one of
its rooms. Any monster you kill on or right next to it counts as a
sacrifice, and now and then your god answers: a little healing, a
recharge for nearby towers, a gift of gold, a burst of speed for your
towers — or just a vague feeling. After an answer the god goes quiet for
a while. Routing monsters past the altar with your maze is the trick.

## Monsters

Between waves, the bottom line tells you what you hear coming ("You hear
the howling of wolves", "You smell brimstone"), so you can build for the
next wave. Before wave 10 it names the dragon's color and what it's
immune to.

Around 30 monster types, in tiers that phase in and out with dungeon
depth — a grid bug that harasses you on Dlvl 1 is long gone by Dlvl 3,
and things like the troll chieftain or kraken don't show up before Dlvl
5. Several glyphs cover more than one tier, real-NetHack-style (the `d`
canine line runs jackal → wolf → warg as you descend, `o` runs orc →
orc captain, `T` runs troll → troll chieftain, `&` runs demon → horned
devil) — same letter, different name/color/stats, so don't assume a
glyph always means the exact same monster.

Special abilities are spread across several tiers rather than one
signature monster apiece, so which ones you'll face varies run to run:
**flying** (cave bat, vampire bat, wraith — ignores your towers
entirely, the counter to turtling behind a wall, and passes over fire
traps untouched too, same reasoning), **submerging** (giant
eel, kraken — periodically untargetable by everything except a scroll
of fireball, and always electricity-immune), **splitting** (demon,
horned devil — spawns a weaker copy of itself on death, and is always
fire-immune), **fast regeneration** (troll, troll chieftain, ghoul,
lich — leave one half-dead and it won't stay that way for long),
**tower-smashing** (orc captain, troll/troll chieftain, ogre, horned
devil — will destroy any tower it stands next to, given time), and
**trap-disarming** (gnome, dwarf — will disable a fire trap they're
standing on instead of triggering it). `D` the dragon is always the
wave-10 boss, tougher on every dungeon level, and its color (and
matching elemental immunity) is re-rolled fresh each level — see below.

**Click any monster to read its lore** — name, stats, special
abilities, and a line of NetHack-style flavor text, including which
element the current dragon resists. Attacking a monster with damage
it's immune to shows a message the first time it happens ("The fire
bounces off its thick scales!") so you're not left guessing why your
damage numbers stopped moving.

**Gaining experience:** killing monsters earns XP (same amount as their
gold bounty) toward your **player level** (no cap), shown as `Lv:N` in the
status bar. Player level, not gold, is the real cap on how far you can
enchant a tower — reaching a new one is announced with its own
pause-for-SPACE message, same treatment as a boss kill. The XP needed
per level grows steeply (roughly 2.3–3x each time) and never stops,
so towers can be enchanted as high as you can level — forever.

**Deeper than Dlvl 5**, monsters keep getting tougher, but so do your
rewards: kills pay 10% more per level past 5. And **greater monsters**
start appearing — highlighted with a gold background, a tougher version
of a normal monster with one ability its kind doesn't usually have
(flying, fast regeneration, or tower-smashing). They're worth double.
More of them show up the deeper you go.

Every monster heals a little when left alone — like a tower recharging,
chip damage that isn't followed up gets undone over time. It's slow
enough that any real sustained fire still wins the race; the troll is
just the one that heals noticeably faster than everything else.

**Each level's dragon is a random color, and every color is immune to
one element:** red/fire, green/acid, blue/electric, white/cold. You
find out which by the reveal message the instant it spawns. A hallway
lined entirely with one damage type won't reliably beat every dragon —
keep more than one type of tower in rotation. The Archer's plain bolts
are untyped and always land regardless of color, which is exactly why
it's worth keeping around even once you've unlocked fancier towers.

## High scores and bones

Your best ten runs are kept in the browser, and the title screen shows
the top three. Die on Dlvl 2 or deeper and you sometimes leave bones: a later run
that reaches that level may meet your ghost partway through, holding the
consumables you never used. Kill it to get them back.

## Controls

Bindings follow NetHack conventions wherever one exists. There's no
"select a tool, then click" step — **click a tile (or move the keyboard
cursor onto it and press `ENTER`) and a menu pops up right there**
listing everything legal on that specific tile: which towers you can
build, or an existing tower's current stats plus upgrade/sell. Press the
`[key]` shown next to an option, or click that option's line directly.
Anything you can't currently afford (or, for enchanting, don't have the
player level for) is greyed out at a glance, so you don't have to read
every price to see what's actually on the table right now. Opening the
menu (or the inventory/help/log) automatically pauses a running wave;
closing it resumes automatically, so you're never racing the clock
while deciding what to build.

- **Mouse** — click a tile to open its menu, click an option line to run
  it, click elsewhere to switch tiles, click the same tile again (or
  `Escape`) to close it. Click a monster instead of a tile to read its
  lore.
- `hjkl`/`yubn`/arrows move a keyboard cursor, `ENTER` opens/closes the
  menu for the cursor's tile — full keyboard play, no mouse required.
- While a menu is open: press the number shown for a build option, `u`
  to upgrade, `s` to sell. Upgrading and selling leave the menu open
  (upgrade again immediately, or see rebuild options right after a
  sale); building closes it.
- `SPACE` — starts the wave while building; pauses/resumes time once a
  wave is underway (a `--More--` prompt, NetHack's own wording, means
  the game is waiting on you specifically — wave-clears, boss kills, and
  player level-ups all stop for it). Also closes the inventory/log/help
  overlay or the monster lore popup, same as `Escape`.
- `r` read scroll · `q` quaff potion · `z` zap wand of cold · `c` zap
  wand of charging · `i` inventory. All three consumables are aimed —
  each arms a targeted deploy, then a click or `ENTER` picks the spot —
  see Items above.
- The inventory screen is directly usable, not just a readout: click an
  item's row (or press its hotkey while the screen is open) to use it on
  the spot instead of closing the menu and pressing the hotkey again.
- `Ctrl-P` message log (like NetHack's `^P`) · `?` help · `Escape` closes
  whatever overlay or menu is open
- `g` toggles box-drawing walls (NetHack's DECgraphics) vs. classic `|` and `-`
- `m` toggles sound (synthesized blips, nothing downloaded)
- `a` auto-start waves · `+`/`-` game speed · `R` restart

Good luck. The dungeon is counting on you.
