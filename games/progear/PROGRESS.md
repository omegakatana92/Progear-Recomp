# Progear: progress

What is known of *Progear* (Europe 010117, the MAME set `progear`)'s memory, and what has been built on
it. Every address here is found by experiment on the running board with the toolkit's own tools
(`cps2recomp/docs/FEATURES.md`); nothing is a dump of the game's data, and no game data is written down.

## The set, and how it was identified

The port is made from MAME's `progear` — *Progear (Europe 010117)*, that set's parent. It was told apart
from the other revisions by the zip's own files: the program ROMs `pgae.03/04` and the key's CRC-32
(`eee6b2a8`) are the European ones (the Japanese set uses `pgaj.*`, the American `pgau.*`, the Asian
`pgaa.*`), while the graphics and QSound ROMs (the `pga-simm.*` files) are shared by all of them.

`games/progear/game.cfg` lists every file with its CRC-32; the port checks them one by one, and **only
this revision is accepted**.

## The graphics layout (a toolkit change)

Progear's graphics ROMs are wired as **eight byte lanes** (the SIMM modules): eight 2 MiB files fill a
64-bit word one byte at a time. Every other CPS-2 game the toolkit has been used for is wired as four
16-bit lanes, which is all `machine/romset.cpp` could lay out. The toolkit copy here adds the eight-byte
form (game.cfg's graphics list begins with `byte `); see `cps2recomp/PATCH.md`. Without it the tiles
would be scrambled.

## The boot self-check (a toolkit change)

The game's start-up code (`0x0264`: `lea $FF0400,a3`, `0x026A`: `lea $0.w,a2`, `0x026E`: `moveq
#0x7,d6`) walks the **whole 4 MB program window** eight times, once per `0x80000` bank: an inner loop
(`0x027C`) reads one byte every `0xFFF` and adds it to a running sum, and the eight 16-bit sums are
written one after another from `0xFF0400` (`0x028A`: `move.w d0,(a3)+`). The board answers those reads
from the program ROM and, past its end, from a region cleared to zero.

Progear's ROM is only 1 MiB, but the 68000's program window is 4 MB. The stock toolkit's `Read16`/`Read8`
indexed the loaded ROM for every address below `0x400000` without checking, so the port died on the first
read past the ROM (`0x100fff`). `machine/machine.cpp` now returns the loaded byte inside the ROM and
**0** outside it — the value MAME gives there (`romload.cpp` zero-fills an unfilled region). See
`cps2recomp/PATCH.md`.

## Verified

- The port **boots and runs**: the QSound/Capcom notice, the title with "INSERT 2 COINS" and the score
  table, and attract mode's DEMONSTRATION play. Seen in headless pictures (`--shot-every`), and the
  graphics come out right — the eight-byte-lane layout and the per-`0x200000` unshuffle agree with
  MAME's `cps2_state::cps2_gfx_decode`.
- 68000 program: **77,994 instructions** (302,710 bytes; the program ROM is 1 MiB) translated to C++ in
  **15,878 blocks** and **2,685 routines** across 48 files. **3,018 entries** came from made-up play
  (`tools/explore.sh`, ten minutes over four rounds), and **80 instructions** still fall back to the
  interpreter (0.1%).
- The Z80 (the QSound sound driver): **3,646 instructions** in 751 blocks and 46 routines; **none left
  to the interpreter**.
- `make check-toolkit` and `make check-common` pass: `cps2recomp/` is the copy `sync-toolkit.sh`
  recorded (with the two changes in `cps2recomp/PATCH.md`, `cps2recomp/COMMIT`), and `recomp-common/` is
  untouched.
- `make check` — translated code against the interpreter, frame by frame, and two runs alike, over
  3,000 frames of made-up play — passes, as does `make check-blocks` (13,573,614 translated blocks
  beside the interpreter).

## The memory so far

| Address | Size | What | How it was found |
| --- | --- | --- | --- |
| `0xFF0000` | 64 KiB | the 68000's work RAM: cleared at boot (`0x13A`, `movea.l #$FF0000,a0` … `move.l d1,(a0)+`, 16,384 longs) | the start-up code, read from `work/progear/program.lst` |
| `0xFF0400` | 16 bytes | eight 16-bit ROM sums, written by the boot self-check and used as scratch afterwards | the start-up loop at `0x0270` (above) |
| `0x804000`+ | — | the board's registers (the custom chip): written throughout start-up | the start-up code |

No game state (credits, lives, score, stage, enemy tables) has been mapped yet — that is the next job, and
the entries above describe the board and the boot, not the game.

## Features built on it (`src/`)

| Feature | Category | What it does |
| --- | --- | --- |
| _(none yet)_ | | |

## Next

- A **coin** first, so a game can be started with no hand (the free-play switch).
- The **player's ship**: position, lives, bombs, the gun/missile power levels.
- The **enemy and bullet tables**, for a hit-box display and a practice mode.
