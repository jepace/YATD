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

- **Mouse** — hover to preview range, click a floor tile `.` to build the
  selected tower, click a tower to inspect/upgrade/sell.
- `1`–`4` select tower type · `SPACE` start wave · `u` upgrade · `s` sell
- `hjkl`/arrows + `ENTER` — full keyboard play, roguelike style
- `a` auto-start waves · `+`/`-` game speed · `?` help · `R` restart

Good luck. The dungeon is counting on you.
