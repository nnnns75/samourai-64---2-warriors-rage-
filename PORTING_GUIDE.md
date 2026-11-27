# Porting guide: recompiling a Hyper Neo Geo 64 game

The method used for Samurai Shodown 64 and for Warriors Rage (`sams64_2`), step by step, with the traps met on the
way. The result of each step is something you can check before moving on. Commands are for Linux (Ubuntu 22.04);
`$REPO` is your clone. Everything derived from the ROM stays on your machine.

## 0. What you need
- Your ROM set (`<game>.zip`) and `hng64.zip` (BIOS + `tmp87ph40an.bin`, the I/O board firmware).
- N64Recomp (commit `598ce20`) built once, Python 3.11, llvm-mingw (downloaded by the build script), Wine to test.
- The MAME definition of the set: `ROM_START( <game> )` in `src/mame/snk/hng64.cpp`. It gives every file's name, size,
  CRC and **where it goes**. Copy that block; do not guess the layout.

## 1. Unpack the ROMs (check: "OK: N fichiers")
Copy `tools/sams64_2/unpack_roms_sams64_2.py` and replace `GAME_ROMS` with the (name, size, CRC-32) of your set (one
copy of each texture: MAME repeats them 4 times). Files are found by size and CRC, so the zip's naming does not matter.

## 2. First analysis
```bash
python3 tools/sams64_2/analyze_hng64_game.py --game <game>.zip --out work --name <game>
```
It reads the header ("HYPER NEOGEO64", entry point 0x80001000), interleaves the first two program ROMs, estimates the
load base from `jal` targets and where the code ends. These are estimates: step 3 gives the truth.

## 3. Decode the startup code (crt0)
```bash
python3 tools/sams64_2/analyze_hng64_game.py --game <game>.zip --out work --name <game> --list 0x200 150
```
Read the listing at ROM 0x200 (= 0x80001000). Write down, from the `lui`/`addiu` pairs:
- `sp` and `gp` (the two first instructions);
- each copy loop: source (0xA4xxxxxx = cart ROM, **subtract 0xA4000000 for the ROM offset**), destination, size;
- the zero-fill loop;
- the vector stubs (`lui k0 / ori k0 / jr k0`): they tell where the game's **exception handler** is;
- the final `lw v0, off(gp); jalr v0`: the word it loads is `main`.
Check the map with a statistic: for the right mapping, most `jal` targets land on function prologues
(`addiu sp, sp, -N`). Trap: a `lui 0xA401` + `addiu -19120` is 0xA400B550, not 0xA401B550 (do the signed arithmetic).

## 4. Extraction script (`extract_<game>.py`)
Declare the segments, `gp`, `main`, the code range and a list of sanity checks (instructions at fixed ROM offsets, so
another revision is refused). It writes the memory image (code + data, holes filled with zeros, so that jump tables
are found), the code, the startup stub, `functions_*.txt` (function seeds), `kernel_candidates.txt` (functions using
CP0 or `cache` instructions) and `segments.json`.
Check: "main = ... present among the seeds", and the list of kernel candidates (expect: exception handler, TLB write,
cache functions).

## 5. Sections, config and translation
- `sections_<game>.py` (installed as `tools/sections.py`): the `.text` image/range and the `.stub` range.
- `<game>.toml`: `ignored = [ ...kernel functions... ]`, `uses_mips3_float_mode = true`.
- `python3.11 tools/sams64_2/translate_<game>.py --fork . --game <game>.zip --n64recomp ~/N64Recomp`.
Errors met and their fixes:
- `Invalid FP control register for cfc1: 0`: the translator refuses to read FIR. Patch that instruction in the
  **translated copy** (`cfc1 a1,$0` becomes `cfc1 a1,$31`); see `PATCHES` in the extraction script.
- Jump tables not recognised (`[Info] Indirect tail call`, or `[FATAL] indirect call to ...` at run time): list them with
  an audit and add a switch fix; add unseen function starts to `EXTRA_STARTS`.

## 6. Adapt the runtime
- **ROM loading** (from MAME's block): program pairs in `hw.cpp`, scroll and sprite tiles in `video.cpp` (sizes
  and offsets differ between games: 32 MB each for the first game, 64 MB for Warriors Rage), textures and vertices in
  `video3d.cpp`, samples in `sound.cpp`. All use `roms/<game>/`.
- **Kernel functions**: identify each ignored function by its disassembly and write its replacement in
  `kernel_<game>.cpp` (TLB write: Index, PageMask, EntryHi, EntryLo0/1, `tlbwi`; cache functions: nothing to do).
  Give the exception handler a stub that aborts if called, because the function table references it.
- **Interrupts**: check that the handler's interrupt path is the same SDK code (IRQ level word at 0xBF701104, handler
  table at 0x80000400, counters at 0x80000460, acknowledge at 0xBF70111C). If so, `sams_irq_dispatch` is reused as is.
- **Vblank status**: register 0x20208048 must read 0xFFFFFFFF during vblank (otherwise the game never uploads its
  palettes: everything stays black).
- `MEM_WU` (`lwu`) must exist in `hw_mem.h` and `recomp_mem_section.txt`.
- **3D windows**: a full-size projection window shifted vertically (512 x 448) is a frustum, not a viewport; only a
  smaller window is a viewport.

## 7. Compile
```bash
python3.11 tools/sams64_2/build_sams64_2_linux.py --fork . --n64recomp ~/N64Recomp
```
Traps: a stale `kernel_*.cpp` or its object in `build/obj/` gives "duplicate symbol"; delete both.

## 8. First boot and debugging
```bash
xvfb-run -a wine build/samurecomp.exe --headless --no-sound --frames 600 --roms roms > run.log 2>&1
grep -n "FATAL\|\[boot\]" run.log
```
Then a script that inserts a coin and presses start, dumping frames:
`--input-script "1800=coin1;1804=;2100=p1start;2104=" --dump-dir frames --dump-frames 2000,6000`.
Reading the signs: `[FATAL] can't open roms/...` is a wrong file name; `[FATAL] indirect call to X` is a missing
function start or jump table; a black screen means the palettes were not loaded (step 6); a picture cut at some
line means a 3D window problem (`BURIKI_LOG3D=1` prints the projection packets). Diagnostic scripts (watch,
coverage, profile of the busiest functions) are described in `docs/NOTES-first-game-samurai-shodown-64.md`.

## 9. Before publishing
- Never publish ROMs, `recomp/generated/`, `work/`, `roms/`, `build/` (the program contains SNK's recompiled code),
  or artwork from the game.
- Keep `LICENSE.md` (ReBuriki's MIT notice) and `licenses/`.
- Rename the interface (window title, menu, messages) and neutralise ReBuriki features that do not apply
  (online play, "Unlock Silber").

## What changes from one game to another
| Item | Samurai Shodown 64 | Warriors Rage |
|---|---|---|
| `gp` | 0x8006CA90 | 0x802F7FD8 |
| Game code | 0x800C0000..0x8015EB00 | 0x80100000..0x8019ECF0 |
| `main` | 0x800CEF50 | 0x8011D130 |
| Exception handler | copied to 0x80000800 | func_80100000 (vectors jump to it) |
| Program ROM | 2 pairs | 4 pairs |
| Scroll / sprite tiles | 32 MB / 32 MB | 64 MB / 64 MB |
| Sound samples | 3 x 4 MB | 4 x 4 MB |
