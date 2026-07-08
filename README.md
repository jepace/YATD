# YATD — Yet Another Tower Defense

A NetHack-flavored, ASCII tower defense game that runs entirely in your
browser and looks like it's playing on a green-phosphor CRT terminal.

**Play it:** just open `index.html` in any modern browser. No build step,
no dependencies, no network — the whole game is one file.

```
python3 -m http.server   # or double-click index.html
```

## The pitch

Monsters pour out of the up staircase `<` and march the winding corridors
of the Dungeons of Doom toward the down staircase `>`. Every monster that
descends costs you HP. Conjure towers on room floors, enchant them, and
survive all 10 waves — the last one brings a red dragon.

Like NetHack, **every game is different**: the dungeon layout is
procedurally generated, wave compositions are randomized, and monsters
randomly drop items. Add `?seed=N` to the URL to replay a particular
dungeon (the seed is shown on the title screen).

## Items

Slain monsters sometimes drop loot: piles of zorkmids, `?` scrolls of
fireball (`r`ead one to scour every monster with flame), `!` potions of
haste (`q`uaff to double your towers' speed for 8s), and `/` wands of
cold (`z`ap to slow everything on the level). Press `i` to check your
pack.

## Towers

| Key | Glyph | Tower            | Cost | What it does                          |
|-----|-------|------------------|------|---------------------------------------|
| `1` | `W`   | Wand of Striking | $10  | Fast single-target bolts               |
| `2` | `%`   | Acid Blob        | $25  | Lobbed splash damage                   |
| `3` | `F`   | Frost Fountain   | $20  | Chills and slows monsters in range     |
| `4` | `^`   | Fire Trap        | $15  | Burns monsters — place ON the corridor |

Towers can be enchanted twice (`u`) for more damage and range, or sold
(`s`) for a 70% refund.

## Monsters

`r` sewer rat · `k` kobold · `a` giant ant · `o` orc · `Z` zombie ·
`T` troll (regenerates!) · `D` red dragon (the boss)

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
