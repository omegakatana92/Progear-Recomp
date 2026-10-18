# cps2recomp: local patches (Progear)

This copy of the cps2recomp toolkit is byte-for-byte upstream **except for the two changes below**, which
are needed to port Progear. See `COMMIT` for the base commit and the file hashes.

## 1. Eight-file (byte-lane) graphics

CPS-2 graphics ROMs are normally wired four files side by side (16-bit lanes): `machine/romset.cpp`
reads four files at a time and makes a 64-bit word from two bytes of each (`ROM_LOAD64_WORD` in MAME).

Progear's board wires its SIMM graphics ROMs as **eight byte lanes**: a 64-bit word is made from one byte
of each of eight files (`ROM_LOAD64_BYTE` in MAME). The stock loader cannot lay this out.

The change:

- `machine/romset.h`: `RomList` gains `bool graphicsBytes`.
- `machine/romset.cpp`: if the graphics list begins with `byte `, `graphicsBytes` is set and
  `LoadRomSet` reads files eight at a time, placing byte *i* of the eight files at graphics offset
  `i*8 + lane` (lane 0..7).

`games/progear/game.cfg` uses it:

```
graphics = byte pga-simm.01c:... pga-simm.01d:... pga-simm.01a:... pga-simm.01b:... \
           pga-simm.03c:... pga-simm.03d:... pga-simm.03a:... pga-simm.03b:...
```

Order matches MAME's `ROM_START(progear)` byte lanes 0..7. The rest of the graphics pipeline (the
per-`0x200000` bank unshuffle, `cps2_state::cps2_gfx_decode`) is unchanged and matches MAME.

## 2. Reads through the whole 4 MB program window

The 68000's program window on CPS-2 is 4 MB (`CODE_SIZE`, `map(0x000000, 0x3fffff).rom()`), but a game's
ROM only fills the first part of it. `Machine::Read16`/`Read8` indexed the loaded ROM for every address
below `0x400000` without checking, so a read past the end ran off the vector's allocation.

Progear reads the whole window: at startup it sums one byte out of every `0x1000` of each `0x80000` bank,
over all eight banks (`games/progear/PROGRESS.md`), and the game's ROM is only 1 MiB. The port died on the
first read past it (address `0x100fff`).

The change (`machine/machine.cpp`):

- `Read16`/`Read8`, for `address < 0x400000`, return the loaded byte when the address is inside the ROM,
  and **0** when it is not — the value MAME gives there, since an unfilled region is cleared to zero
  (`romload.cpp`: `memset(region, ROMREGION_GETERASEVAL(region), ...)`, else zero).

The two fetch paths already answered for addresses past the ROM (`Machine::Fetch16` falls back to
`Read16`, and `runtime/native.h`'s `P16` to `Fetch16`), so this only closes the data-read gap.

Nothing else differs from upstream. Both changes are generic (any eight-lane set, any game whose program
ROM is smaller than the window) and are candidates for upstreaming to cps2recomp.
