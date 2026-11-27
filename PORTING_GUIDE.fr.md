# Guide de portage : recompiler un jeu Hyper Neo Geo 64

La méthode utilisée pour Samurai Shodown 64 et pour Warriors Rage (`sams64_2`), étape par étape, avec les pièges
rencontrés. Chaque étape donne un résultat à vérifier avant de passer à la suivante. Les commandes sont pour Linux
(Ubuntu 22.04) ; `$REPO` est votre copie du dépôt. Tout ce qui dérive de la ROM reste sur votre machine.

## 0. Ce qu'il faut
- Votre set de ROMs (`<jeu>.zip`) et `hng64.zip` (BIOS + `tmp87ph40an.bin`, le firmware de la carte d'E/S).
- N64Recomp (commit `598ce20`) compilé une fois, Python 3.11, llvm-mingw (téléchargé par le script de compilation), Wine pour tester.
- La définition MAME du set : `ROM_START( <jeu> )` dans `src/mame/snk/hng64.cpp`. Elle donne le nom, la taille, le CRC
  de chaque fichier et **où il va**. Copiez ce bloc ; ne devinez pas la disposition.

## 1. Extraire les ROMs (vérif. : « OK: N fichiers »)
Copiez `tools/sams64_2/unpack_roms_sams64_2.py` et remplacez `GAME_ROMS` par les (nom, taille, CRC-32) de votre set (une
seule copie de chaque texture : MAME les répète 4 fois). Les fichiers sont retrouvés par taille et CRC : le nom dans le zip importe peu.

## 2. Première analyse
```bash
python3 tools/sams64_2/analyze_hng64_game.py --game <jeu>.zip --out work --name <jeu>
```
Il lit l'en-tête (« HYPER NEOGEO64 », point d'entrée 0x80001000), entrelace les deux premières ROMs de programme,
estime l'adresse de chargement d'après les cibles de `jal` et la fin du code. Ce sont des estimations : l'étape 3 donne la vérité.

## 3. Décoder le code de démarrage (crt0)
```bash
python3 tools/sams64_2/analyze_hng64_game.py --game <jeu>.zip --out work --name <jeu> --list 0x200 150
```
Lisez le listing à partir de la ROM 0x200 (= 0x80001000). Notez, d'après les paires `lui`/`addiu` :
- `sp` et `gp` (les deux premières instructions) ;
- chaque boucle de copie : source (0xA4xxxxxx = ROM cartouche, **soustraire 0xA4000000 pour l'offset ROM**), destination, taille ;
- la boucle de remise à zéro ;
- les vecteurs (`lui k0 / ori k0 / jr k0`) : ils indiquent où est le **gestionnaire d'exceptions** du jeu ;
- le `lw v0, off(gp); jalr v0` final : le mot chargé est `main`.
Vérifiez la carte par une statistique : avec la bonne carte, la plupart des cibles de `jal` tombent sur des débuts de
fonction (`addiu sp, sp, -N`). Piège : `lui 0xA401` + `addiu -19120` vaut 0xA400B550, pas 0xA401B550 (calcul signé).

## 4. Script d'extraction (`extract_<jeu>.py`)
Déclarez les segments, `gp`, `main`, la plage de code et des contrôles (instructions à des offsets ROM fixes, pour
refuser une autre révision). Il écrit l'image mémoire (code + données, trous remplis de zéros pour que les tables de
sauts soient trouvées), le code, l'amorce, `functions_*.txt` (graines de fonctions), `kernel_candidates.txt` (fonctions
qui utilisent CP0 ou `cache`) et `segments.json`.
Vérif. : « main = ... présent parmi les graines », et la liste des candidats noyau (attendu : gestionnaire d'exceptions,
écriture TLB, fonctions de cache).

## 5. Sections, configuration et traduction
- `sections_<jeu>.py` (installé comme `tools/sections.py`) : image et plage de `.text`, plage de `.stub`.
- `<jeu>.toml` : `ignored = [ ...fonctions noyau... ]`, `uses_mips3_float_mode = true`.
- `python3.11 tools/sams64_2/translate_<jeu>.py --fork . --game <jeu>.zip --n64recomp ~/N64Recomp`.
Erreurs rencontrées et corrections :
- `Invalid FP control register for cfc1: 0` : le traducteur refuse de lire FIR. Corrigez cette instruction dans la
  **copie traduite** (`cfc1 a1,$0` devient `cfc1 a1,$31`) ; voir `PATCHES` dans le script d'extraction.
- Tables de sauts non reconnues (`[Info] Indirect tail call`, ou `[FATAL] indirect call to ...` à l'exécution) : les
  lister avec un audit et ajouter une correction de switch ; ajouter les débuts de fonction manquants à `EXTRA_STARTS`.

## 6. Adapter le runtime
- **Chargement des ROMs** (d'après le bloc MAME) : paires de programme dans `hw.cpp`, tuiles de décors et de sprites dans
  `video.cpp` (tailles et offsets différents selon le jeu : 32 Mo chacun pour le premier jeu, 64 Mo pour Warriors
  Rage), textures et vertex dans `video3d.cpp`, échantillons dans `sound.cpp`. Tout lit `roms/<jeu>/`.
- **Fonctions noyau** : identifier chaque fonction ignorée par son désassemblage et écrire son remplacement dans
  `kernel_<jeu>.cpp` (écriture TLB : Index, PageMask, EntryHi, EntryLo0/1, `tlbwi` ; fonctions de cache : rien à faire).
  Donner au gestionnaire d'exceptions une fonction vide qui s'arrête si on l'appelle, car la table des fonctions y fait référence.
- **Interruptions** : vérifier que le chemin d'interruption du gestionnaire est le même code SDK (niveau d'IRQ en
  0xBF701104, table des gestionnaires en 0x80000400, compteurs en 0x80000460, acquittement en 0xBF70111C). Si oui,
  `sams_irq_dispatch` est réutilisé tel quel.
- **État vblank** : le registre 0x20208048 doit valoir 0xFFFFFFFF pendant le vblank (sinon le jeu ne charge jamais ses
  palettes : tout reste noir).
- `MEM_WU` (`lwu`) doit exister dans `hw_mem.h` et `recomp_mem_section.txt`.
- **Fenêtres 3D** : une fenêtre de projection pleine taille décalée verticalement (512 x 448) est un frustum, pas une
  fenêtre d'affichage ; seule une fenêtre plus petite en est une.

## 7. Compiler
```bash
python3.11 tools/sams64_2/build_sams64_2_linux.py --fork . --n64recomp ~/N64Recomp
```
Pièges : un ancien `kernel_*.cpp` ou son objet dans `build/obj/` donne « duplicate symbol » ; supprimez les deux.

## 8. Premier démarrage et débogage
```bash
xvfb-run -a wine build/samurecomp.exe --headless --no-sound --frames 600 --roms roms > run.log 2>&1
grep -n "FATAL\|\[boot\]" run.log
```
Puis un script qui insère une pièce et appuie sur start, en sortant des captures :
`--input-script "1800=coin1;1804=;2100=p1start;2104=" --dump-dir frames --dump-frames 2000,6000`.
Lire les signes : `[FATAL] can't open roms/...` = mauvais nom de fichier ; `[FATAL] indirect call to X` = début de
fonction ou table de sauts manquant ; écran noir = palettes non chargées (étape 6) ; image coupée à une ligne = problème
de fenêtre 3D (`BURIKI_LOG3D=1` affiche les paquets de projection). Les scripts de diagnostic (surveillance de
variables, couverture, profil des fonctions les plus appelées) sont décrits dans `docs/NOTES-first-game-samurai-shodown-64.md`.

## 9. Avant de publier
- Ne publiez jamais ROMs, `recomp/generated/`, `work/`, `roms/`, `build/` (le programme contient du code SNK
  recompilé), ni illustrations du jeu.
- Gardez `LICENSE.md` (mention MIT de ReBuriki) et `licenses/`.
- Renommez l'interface (titre de fenêtre, menu, messages) et neutralisez les fonctions de ReBuriki qui ne
  s'appliquent pas (jeu en ligne, « Unlock Silber »).

## Ce qui change d'un jeu à l'autre
| Élément | Samurai Shodown 64 | Warriors Rage |
|---|---|---|
| `gp` | 0x8006CA90 | 0x802F7FD8 |
| Code du jeu | 0x800C0000..0x8015EB00 | 0x80100000..0x8019ECF0 |
| `main` | 0x800CEF50 | 0x8011D130 |
| Gestionnaire d'exceptions | copié en 0x80000800 | func_80100000 (les vecteurs y sautent) |
| ROM programme | 2 paires | 4 paires |
| Tuiles décors / sprites | 32 Mo / 32 Mo | 64 Mo / 64 Mo |
| Échantillons sonores | 3 x 4 Mo | 4 x 4 Mo |
