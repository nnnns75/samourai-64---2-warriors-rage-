<div align="center">

# SamuRecomp 2

**Samurai Shodown 64 2: Warriors Rage (Samurai Spirits 2: Asura Zanmaden), rebuilt as a native program.**<br>
An unofficial port of SNK's 1998 Hyper Neo Geo 64 arcade fighter, built from the arcade ROMs you own.

[Français](README.fr.md) · [Build guide](docs/BUILD.md) · [Status](#status) · [Credits](#credits-and-licences)

</div>

> [!IMPORTANT]
> SamuRecomp 2 does **not** include the game, and it never will. You need your own legally obtained copies of two
> arcade ROM sets, as MAME names them: `sams64_2.zip` (Samurai Shodown 64: Warriors Rage) and `hng64.zip` (the
> Hyper Neo Geo 64 system ROMs). The build steps translate the game's program on **your** machine. This repository
> holds only project code and open-source tools. Please do not upload the built program, `recomp/generated/`,
> `work/`, `roms/`, any ROM or any artwork from the game anywhere: they belong to SNK (the `.gitignore` already
> excludes them).

## What it is

Not an emulator: the arcade's MIPS program is translated to C with
[N64Recomp](https://github.com/N64Recomp/N64Recomp) and compiled together with a hand-written model of the
Hyper Neo Geo 64 hardware (video, 3D, sound, I/O board). It is the same approach as
[ReBuriki](https://github.com/QuietClubGames/ReBuriki) for Buriki One, which this project is based on. It is a
sibling of the port of the first game, Samurai Shodown 64 (`sams64`).

## Status

Playable. Tested so far (Wine on a Steam Deck, with a window, and headless):

| Works | To check / not done |
|---|---|
| Boot, warning screen, attract mode, title screen, coin and start | Arcade settings (difficulty, free play, blood...): need this game's save-memory addresses |
| Fighter select, intro, full fights in colour (2D layers, textured 3D fighters and stages, HUD, rage gauge) | Online play and "Unlock Silber" are Buriki One features: disabled |
| Sound | Windows `Setup.exe` installer is not adapted; building is done with the scripts in `tools/sams64_2` |
| Window / borderless / full screen, scaling and filters, keyboard and controller menus | A fighter's head sometimes flashes white in fights: not yet checked against the arcade |
| Optional menu background image (`build/menu_art.png` or `.jpg`, your own file) | The high-resolution 3D option (inherited from ReBuriki) is untested on this game |

Only the `sams64_2` set (MAME `005…` ROMs) is supported. The first game, the other Hyper Neo Geo 64 titles and
other revisions are not.

## Build it

See **[docs/BUILD.md](docs/BUILD.md)**. In short: unpack your ROMs with `tools/sams64_2/unpack_roms_sams64_2.py`,
translate with `translate_sams64_2.py`, compile with `build_sams64_2_linux.py` (Linux, cross-compiling a Windows
program, tested under Ubuntu 22.04), then run it with Wine, Proton or on Windows.

## Controls (defaults)

Player 1: arrows, `Z X C V` = A B C D, `1` start, `5` coin. Player 2: `R F D G`, `A S Q W`, `2`, `6`.
`Esc` / `Tab` menu (on a pad: Back + Start), `F11` full screen, `F9` enhanced / original graphics, `F2` arcade Test
menu, `9` Service. Xbox-style controllers work (remap in the menu).

## How it was ported

To port another game the same way: **[docs/PORTING_GUIDE.md](docs/PORTING_GUIDE.md)** (step by step).

[docs/BOOT_PLAN_sams64_2.md](docs/BOOT_PLAN_sams64_2.md) records the boot layout decoded from the ROM's own startup
code, each problem met and its fix. [docs/NOTES-first-game-samurai-shodown-64.md](docs/NOTES-first-game-samurai-shodown-64.md)
holds the lessons from the first game (including the vblank status register that kept the palettes from loading),
and [docs/PORTING_MAP.md](docs/PORTING_MAP.md) lists what in ReBuriki is specific to Buriki One.

## Credits and licences

- [ReBuriki](https://github.com/QuietClubGames/ReBuriki) by QuietClubGames (MIT): the runtime, front end and build
  chain this project adapts.
- [MAME](https://www.mamedev.org/) (`hng64` driver and I/O board firmware wiring): LGPL-2.1+ / BSD-3-Clause parts
  ported to the runtime, see [LICENSE.md](LICENSE.md) and `licenses/`.
- [N64Recomp](https://github.com/N64Recomp/N64Recomp) (MIT) translates the MIPS code.
- ReLayer (MIT): front-end pieces used by ReBuriki.

Everything is unofficial and not affiliated with SNK. Samurai Shodown, Samurai Spirits and Hyper Neo Geo 64 belong
to their owners.
