<div align="center">

# SamuRecomp 2

**Samurai Shodown 64 2 : Warriors Rage (Samurai Spirits 2 : Asura Zanmaden), reconstruit en programme natif.**<br>
Portage non officiel du jeu de combat d'arcade Hyper Neo Geo 64 de SNK (1998), construit à partir des ROMs que vous possédez.

[English](README.md) · [Guide de compilation](docs/BUILD.md) · [Avancement](#avancement) · [Crédits](#crédits-et-licences)

</div>

> [!IMPORTANT]
> SamuRecomp 2 ne contient **pas** le jeu, et n'en contiendra jamais. Il faut vos propres copies légales de deux
> ROMs d'arcade, nommées comme dans MAME : `sams64_2.zip` (Samurai Shodown 64: Warriors Rage) et `hng64.zip` (les
> ROMs système Hyper Neo Geo 64). Les étapes de compilation traduisent le programme du jeu **sur votre machine**.
> Ce dépôt ne contient que du code de projet et des outils libres. Ne publiez nulle part le programme compilé,
> `recomp/generated/`, `work/`, `roms/`, ni aucune ROM ni illustration du jeu : ils appartiennent à SNK (le
> `.gitignore` les exclut déjà).

## De quoi s'agit-il

Ce n'est pas un émulateur : le programme MIPS de l'arcade est traduit en C avec
[N64Recomp](https://github.com/N64Recomp/N64Recomp), puis compilé avec un modèle écrit à la main du matériel
Hyper Neo Geo 64 (vidéo, 3D, son, carte d'E/S). C'est la même méthode que [ReBuriki](https://github.com/QuietClubGames/ReBuriki)
pour Buriki One, dont ce projet dérive. C'est le projet frère du portage du premier jeu, Samurai Shodown 64 (`sams64`).

## Avancement

Jouable. Testé sur Wine (Steam Deck), avec fenêtre et sans écran :

| Fonctionne | À vérifier / pas fait |
|---|---|
| Démarrage, écran d'avertissement, attract mode, écran-titre, pièce et start | Réglages d'arcade (difficulté, free play, sang...) : il faut les adresses de la mémoire de sauvegarde de ce jeu |
| Sélection des personnages, intro, combats complets en couleur (couches 2D, personnages et décors 3D texturés, interface, jauge de rage) | Jeu en ligne et « Unlock Silber » : fonctions de Buriki One, désactivées |
| Son | L'installateur Windows `Setup.exe` n'est pas adapté ; la compilation passe par les scripts de `tools/sams64_2` |
| Fenêtre / sans bordure / plein écran, échelle et filtres, menus clavier et manette | Une tête de combattant clignote parfois en blanc : pas encore comparée à l'arcade |
| Image de fond du menu en option (`build/menu_art.png` ou `.jpg`, votre propre fichier) | L'option 3D haute résolution (héritée de ReBuriki) n'est pas testée sur ce jeu |

Seul le set `sams64_2` (ROMs MAME `005…`) est géré. Le premier jeu, les autres jeux Hyper Neo Geo 64 et les autres
révisions ne le sont pas.

## Compiler

Voir **[docs/BUILD.md](docs/BUILD.md)** (en anglais). En résumé : extraire vos ROMs avec
`tools/sams64_2/unpack_roms_sams64_2.py`, traduire avec `translate_sams64_2.py`, compiler avec
`build_sams64_2_linux.py` (Linux, compilation croisée d'un programme Windows, testé sous Ubuntu 22.04), puis lancer
avec Wine, Proton ou sous Windows.

## Commandes (par défaut)

Joueur 1 : flèches, `Z X C V` = A B C D, `1` start, `5` pièce. Joueur 2 : `R F D G`, `A S Q W`, `2`, `6`.
`Échap` / `Tab` : menu (manette : Back + Start), `F11` plein écran, `F9` graphismes améliorés / d'origine, `F2` menu
Test de la borne, `9` Service. Les manettes de type Xbox fonctionnent (reconfigurables dans le menu).

## Comment le portage a été fait

Pour porter un autre jeu de la même façon : **[docs/PORTING_GUIDE.fr.md](docs/PORTING_GUIDE.fr.md)** (pas à pas).

[docs/BOOT_PLAN_sams64_2.md](docs/BOOT_PLAN_sams64_2.md) décrit la carte mémoire décodée depuis le code de démarrage de
la ROM, chaque problème rencontré et sa correction. [docs/NOTES-first-game-samurai-shodown-64.md](docs/NOTES-first-game-samurai-shodown-64.md)
garde les leçons du premier jeu, et [docs/PORTING_MAP.md](docs/PORTING_MAP.md) liste ce qui est propre à Buriki One dans ReBuriki.

## Crédits et licences

- [ReBuriki](https://github.com/QuietClubGames/ReBuriki) de QuietClubGames (MIT) : le runtime, l'interface et la
  chaîne de compilation que ce projet adapte.
- [MAME](https://www.mamedev.org/) (pilote `hng64` et câblage du firmware de la carte d'E/S) : parties LGPL-2.1+ /
  BSD-3-Clause portées dans le runtime, voir [LICENSE.md](LICENSE.md) et `licenses/`.
- [N64Recomp](https://github.com/N64Recomp/N64Recomp) (MIT) traduit le code MIPS.
- ReLayer (MIT) : éléments d'interface utilisés par ReBuriki.

Projet non officiel, sans lien avec SNK. Samurai Shodown, Samurai Spirits et Hyper Neo Geo 64 appartiennent à leurs propriétaires.
