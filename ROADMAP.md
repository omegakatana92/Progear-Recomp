# Progear — features and roadmap

Where this port stands against a finished one, and what each missing piece needs. The reference is the
toolkit's own complete example, [vampire-savior-recomp](https://github.com/tmcleroy/vampire-savior-recomp)
(`games/vampire_savior/src/`), and the toolkit's documents `docs/FEATURES.md` and `docs/FIGHT.md`.

Everything below is an **optional feature**: with every feature off, the port must run exactly as the
board does. Nothing from the game (no ROM, no RAM dump, no ripped art/text) may ever be committed — each
thing found is recorded in `PROGRESS.md` as *how* it was found (a RAM diff, a write watch, a trace),
never as the data itself.

Progear is a horizontal bullet-hell shooter, not a fighting game, so the toolkit's `fight/` training mode
does not apply as it stands; what this port wants instead is smaller and its own.

## What works today, with no game work

These come from the toolkit and already run in `build/progear/progear.exe`:

- The **launcher**: chooses the ROM set, checks every file by CRC-32, sets the controls, Play.
- The **in-game menu** (Esc, `F1`, a stick's click, or Start+Select): save states (9 slots), Reset,
  Insert coin, Player 2 joins, Quit game.
- **Stop, step, slow** (P, O, `-`/`=`).
- **Run-ahead** and the **Display** options (scaling, picture shape, window size, full screen).
- Every key and button **remappable**, for each player.

`--list-features` shows these; today they are the only ones.

## What a finished port adds (this is the work)

### 0. Foundation — know the game's memory

Nearly every feature below reads the game's memory or hooks a routine, so the first job is a **RAM map**
for Progear: the credits and coin count, each player's plane position and lives, the jewel counter and
the jewel class, the bombs, the gunner/pilot mode, the score and the stage, the enemy and bullet tables,
and the game/loop state. The names go in a new `games/progear/src/progear.h`, the way `vsav.h` does.

How the toolkit finds these (all documented in `docs/FEATURES.md`):

- `--dump-ram F:FILE` + `tools/ramfind.py DUMP=VALUE ...` — addresses that held a known value across
  several dumps (this is how a credit counter, a score and a lives count are found).
- `--trace LOW-HIGH:FILE` with a movie that does known things at known frames (this is how the plane's
  position and the jewel counter are found).
- `--watch LOW[-HIGH]` — every write to an address and the instruction that made it, read against
  `work/progear/program.lst`.
- `--poke ADDR=BYTE,...` to try a guess; `--shot-every N:PFX` + `tools/sheet.sh` to see a run.

Deliverable: `progear.h` (names) and a `board.cpp` that reads the EEPROM's operator settings.

### 1. The Features page (the game's own switches)

Registered from `games/progear/src/`; they then appear on the launcher's Features page and the in-game menu,
and are kept in `settings.ini`.

| Feature | Kind | Needs |
| --- | --- | --- |
| `free_play` (Enhanced) | on/off | the credit/coin word |
| `infinite_lives`, `no_damage` | on/off | the lives word and the collision/damage routine |
| `infinite_bombs` | on/off | the bomb count |
| `max_jewels` | on/off | the jewel counter and class, so the bomb and gunner stay at their best |
| `max_gunner` | on/off | the gunner's power/relationship level |
| Board/operator settings (difficulty, lives, extend) | choices | reading the EEPROM settings |
| "The game's own test menu" | command | a routine address to jump to |

Quick wins: these prove the RAM map and populate the menu.

### 2. Practice mode (the headline feature for a shooter)

| Piece | What it adds | Needs |
| --- | --- | --- |
| Stage select | start on any of the five stages | the stage id and the routine that loads one |
| Hit-box display | draw the plane's and enemies' collision boxes | the box tables |
| Invincibility toggle | fly through bullets while practising | the damage routine (hook) or the lives word |
| Slow motion (toolkit) | study a pattern | already in the toolkit |
| Save states (toolkit) | retry a pattern from a saved frame | already in the toolkit |
| Input display | what the stick holds, and for how long | the port's panel bits, the plane's facing |

`fight::InputHistory` gives the input display for free once the game's facing is known, the way it did for
Cyberbots.

### 3. Alternative modes

Each is a command with a page of its own:

| Mode | What it adds | Needs |
| --- | --- | --- |
| Attract watcher | the game's own show on a loop, no coin | a credit word to leave at zero |
| Second-loop practice | the second loop's route, without clearing bomb-frugally first | the loop/clear flag |
| Endless | a game that never ends, stage after stage | the stage-advance and end state |

### 4. Widescreen

A switch on the launcher's first page, off in every preset (it can change how the game plays). The
scrolling field is shown in 16:9, 512 columns where the board's 384, at the same height and scale — a
third more of the field (nothing is stretched). Needs rewrites of the camera and the background scrollers,
and the objects that must be kept while they are in the wide picture. Progear's bullet-heavy play means
the extra width changes what can be dodged, so this one is off everywhere by default. The toolkit's
`widescreen.cpp` in Vampire Savior is the example.

### 5. Readable rewrites (correctness, and what the above builds on)

Rewrite the game's key routines by hand as named C++ (`docs/READABLE.md`): the plane's movement and
collision, the bullet/enemy update, the jewel spawning and tractor, damage, the score and the stage
advance. Vampire Savior has 123 routines rewritten; each is checked call by call with `make check-rewrites`.
This is what makes the practice mode and widescreen possible, and it is the part most worth doing carefully.

## Recommended order

1. **Foundation** (`progear.h` RAM map + `board.cpp`) — everything depends on it.
2. **Free play + cheats** — small, proves the map, fills the Features page.
3. **Practice mode** — stage select, hit boxes, invincibility.
4. **Readable rewrites** — needed by 3 and 5, and correctness.
5. **Widescreen.**
6. **Alternative modes.**

**Done so far:** nothing yet — the project has just been created and builds with the toolkit's own
features (`games/progear/`), plus the two toolkit changes for Progear's graphics and its program window.

## Before uploading: a checklist

- [ ] `README.md` states the exact ROM set (`progear`, Europe 010117) and that **only that revision** works,
      with the Capcom disclaimer.
- [ ] `rom/`, `build/`, `work/` are ignored and **absent** from the repository; no game data, keys or dumps
      anywhere; no screenshots beyond the few needed for the README.
- [ ] `PROGRESS.md` records how each finding was made (a RAM diff, a watch, a trace), not the data.
- [ ] `cps2recomp/PATCH.md` is the only change to the toolkit copy, and `make check-toolkit` passes with it
      recorded in `cps2recomp/COMMIT`.
- [ ] `make check`, `make check-toolkit`, `make check-common` all pass.
- [ ] `LICENSE` (GPL-3.0-only) and `THIRD_PARTY_NOTICES.md` (the cipher is MAME's, BSD-3-Clause).
- [ ] Repo description set (see README's title line / the About field).
- [ ] First commit made; the repository builds from a fresh clone with the player's own ROM set.
