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
HP you finished with (plus a small heal for the stairs down). Do that
five times — Dlvl 5 — and you win. Towers don't carry over between
levels, so every floor is a fresh maze to build.

Like NetHack, **every game is different**: every dungeon level is
freshly procedurally generated, wave compositions are randomized, and
monsters randomly drop items. Add `?seed=N` to the URL to replay a
particular first dungeon (the seed is shown on the title screen).

## Items

Slain monsters sometimes drop loot: piles of zorkmids, `?` scrolls of
fireball (`r`ead one to scour every monster with flame), `!` potions of
haste (`q`uaff to double your towers' speed for 8s), and `/` wands of
cold (`z`ap to slow everything on the level). Press `i` to check your
pack.

## Towers

Tower glyphs are always NetHack object-class punctuation, never a
letter — letters are reserved for monsters, so nothing on the map is
ever ambiguous about what it is.

| Key | Glyph | Tower            | Cost | What it does                          |
|-----|-------|------------------|------|---------------------------------------|
| `1` | `/`   | Wand of Striking | $10  | Fast single-target bolts, limited charges |
| `2` | `0`   | Acid Blob        | $25  | Lobbed splash damage                   |
| `3` | `{`   | Frost Fountain   | $20  | Chills and slows monsters in range     |
| `4` | `^`   | Fire Trap        | $15  | Burns monsters — place ON the corridor |

The Wand of Striking runs on charges — it fires until it's dry, then
sits idle (dimmed on the map) recharging slowly on its own. Enchanting
it (`u`) tops it back up and raises its max charges. It's the cheapest
tower for a reason; don't rely on a wall of wands alone to carry a wave.

Every tower except the fire trap needs a clear line of sight — thick
walls block shots, so a tower can't hit monsters marching down the
corridor on the other side of a wall.

The wand, acid blob, and frost fountain physically occupy their tile —
build one in a monster's way and it routes around, same as any wall.
Build a maze if you want, but you can never wall the stairs shut
completely: the game refuses any placement that would leave no route at
all from `<` to `>`. The fire trap is the exception — it lies flush with
the floor, doesn't block anything, and only goes directly on a corridor
or doorway tile where something will actually walk over it.

Towers can be enchanted up to level 4 (`u`) for more damage, range, and
(wands) charges, or sold (`s`) for a 70% refund.

## Monsters

`r` sewer rat · `x` grid bug (weak but fast and cheap) · `a` giant ant ·
`b` cave bat (**flies over your towers** — the counter to turtling
behind a wall) · `k` kobold · `;` giant eel (**submerges** — periodically
untargetable by everything except a scroll of fireball) · `o` orc ·
`&` demon (splits into a weaker copy when killed) · `Z` zombie ·
`T` troll (regenerates!) · `D` red dragon (the boss, tougher on every
dungeon level)

## Controls

Bindings follow NetHack conventions wherever one exists.

- **Mouse** — hover to preview range, click a tower type in the status
  line's tool prompt (or press its number), then click a floor tile `.`
  to build it. Click an existing tower to inspect/upgrade/sell it.
- `1`–`4` ready a tower type — **you must press the number again before
  each placement**, so clicking around never drops a second tower by
  accident.
- `hjkl`/`yubn`/arrows move a keyboard cursor, `ENTER` builds/selects —
  full keyboard play, no mouse required.
- `SPACE` — starts the wave while building; pauses/resumes time once a
  wave is underway.
- `r` read scroll · `q` quaff potion · `z` zap wand · `i` inventory
- `u` upgrade selected tower · `s` sell selected tower
- `Ctrl-P` message log (like NetHack's `^P`) · `?` help · `Escape` closes
  overlays/deselects
- `a` auto-start waves · `+`/`-` game speed · `R` restart

Good luck. The dungeon is counting on you.
