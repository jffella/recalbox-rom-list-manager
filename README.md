# recalbox_favorites

Gestion des favoris EmulationStation / Recalbox via les fichiers `gamelist.xml`.

## Prérequis

- Python 3.9+
- Aucune dépendance tierce (stdlib uniquement)

## Installation

```bash
# Rendre le script exécutable (optionnel)
chmod +x ~/recalbox_favorites.py
```

## Utilisation

```
python3 recalbox_favorites.py SOURCE [options] SOUS-COMMANDE [args]
```

`SOURCE` est le chemin vers le dossier racine des ROMs (ou directement un `gamelist.xml`).

### Options globales

| Option | Description |
|---|---|
| `--threshold PCT` | Seuil de similarité pour la correspondance de noms, en % (défaut : 100 = exact) |
| `-n` / `--dry-run` | Simule toutes les opérations sans rien écrire sur le disque |
| `-v` / `--verbose` | Affiche les messages DEBUG dans la console |
| `--log FILE` | Fichier de log (défaut : `recalbox_favorites.log`) |
| `--v9` | Force le format Recalbox v9 (legacy) — voir section dédiée |

---

## Sous-commandes

### `export` — Exporter les favoris vers un fichier JSON

```bash
python3 recalbox_favorites.py ~/roms export favoris.json
```

Produit un fichier JSON avec tous les jeux marqués `<favorite>1</favorite>` ou
`<favorite>true</favorite>` dans l'arborescence.

```json
[
  {
    "name": "Sonic The Hedgehog",
    "path": "./Sonic The Hedgehog (Europe).md",
    "system": "megadrive",
    "gamelist": "/home/user/roms/megadrive/gamelist.xml"
  }
]
```

---

### `export-text` — Exporter les favoris vers un fichier texte

```bash
python3 recalbox_favorites.py ~/roms export-text favoris.txt
```

Produit un fichier texte lisible, regroupé par système :

```
[Sega Mega Drive]
Sonic The Hedgehog
Streets of Rage 2

[MAME]
Donkey Kong
Final Fight
```

Ce format est directement réutilisable avec la commande `mark`.

---

### `apply` — Appliquer un fichier JSON de favoris

```bash
python3 recalbox_favorites.py ~/roms apply favoris.json
```

Pour chaque entrée du JSON, cherche le jeu dans les `gamelist.xml` et active
`<favorite>`. Utilise le champ `"gamelist"` pour restreindre la recherche au
bon système quand c'est possible.

```bash
# Simulation avant d'appliquer
python3 recalbox_favorites.py -n ~/roms apply favoris.json
```

---

### `mark` — Marquer des favoris depuis un fichier texte

```bash
python3 recalbox_favorites.py ~/roms mark favoris.txt
```

**Format du fichier texte :**

```
# Commentaire ignoré

[Mega Drive]
Sonic The Hedgehog
Streets of Rage 2

[MAME]
Donkey Kong
Final Fight

# Sans section = recherche dans tous les systèmes
Tetris
```

Les en-têtes de section acceptent les noms longs (`[Sega Mega Drive]`), les alias
(`[Genesis]`) et les noms de dossiers (`[megadrive]`). La casse est ignorée.
Voir [Annexe — Alias de systèmes acceptés](#annexe--alias-de-systèmes-acceptés) pour la liste complète.

#### Option `--by-rom` — Correspondance sur le nom de fichier ROM

```bash
python3 recalbox_favorites.py ~/roms mark --by-rom favoris-roms.txt
```

La liste contient des stems de fichiers ROM (sans extension) au lieu des noms de jeux.

```
[mame]
3wonders
bbmanw
dkong

[megadrive]
sonic
s2
```

La correspondance est exacte sur le champ `<path>` du gamelist :
`3wonders` → `./3wonders.zip`.

#### Option `--threshold` — Correspondance approximative

```bash
# Tolère les légères différences de titre (sous-titres, accents…)
python3 recalbox_favorites.py --threshold 85 ~/roms mark favoris.txt
```

Incompatible avec `--by-rom` (qui est toujours exact).

---

### `unmark` — Retirer tous les favoris

```bash
python3 recalbox_favorites.py ~/roms unmark
```

Met `<favorite>0</favorite>` pour tous les jeux marqués dans l'arborescence.

```bash
# Simulation
python3 recalbox_favorites.py -n ~/roms unmark
```

---

## Compatibilité Recalbox v10

À partir de la version 10, Recalbox stocke ses métadonnées utilisateur (favoris,
compteurs de jeu, date de dernière partie) dans `gamelist-userdata.ini`, placé
dans le même répertoire que `gamelist.xml`. Le fichier `gamelist.xml` n'est jamais
modifié en mode v10.

### Comportement par défaut (v10)

Par défaut, le script opère en **mode v10** : il lit et écrit les favoris dans
`gamelist-userdata.ini` ; `gamelist.xml` n'est jamais modifié.

### Option `--v9` — Format Recalbox v9 (legacy)

```bash
python3 recalbox_favorites.py --v9 ~/roms mark favoris.txt
```

Force l'utilisation de `gamelist.xml` pour toutes les opérations (ancien comportement).

```bash
# Simulation
python3 recalbox_favorites.py --v9 -n ~/roms mark favoris.txt
```

### `unmark` en mode v10 — Supprimer les entrées favorites

```bash
python3 recalbox_favorites.py ~/roms unmark
```

En mode v10 (défaut), `unmark` retire la clé `favorite` de chaque entrée dans
`gamelist-userdata.ini` (et supprime le fichier si c'était la seule entrée).

### Format gamelist-userdata.ini

```
galaga.zip:favorite=true
rtype.zip:timeplayed=138,lastplayed=20260608T191434,playcount=1,favorite=true
mame0274/xybots.zip:favorite=true
```

Le chemin ROM est le champ `<path>` du `gamelist.xml` sans le préfixe `./`.
Les champs sont ordonnés : `timeplayed`, `lastplayed`, `playcount`, `favorite`.

---

## Exemples complets

### Workflow standard (v10) : sauvegarde et restauration des favoris

```bash
# 1. Exporter les favoris actuels
python3 recalbox_favorites.py ~/roms export sauvegarde.json

# 2. (après réinstallation) Restaurer
python3 recalbox_favorites.py ~/roms apply sauvegarde.json
```

### Workflow standard (v10) : initialiser les favoris depuis une liste de ROMs

```bash
# 1. Marquer les favoris
python3 recalbox_favorites.py ~/roms mark --by-rom favoris-roms.txt

# 2. Tout effacer
python3 recalbox_favorites.py ~/roms unmark
```

### Workflow v9 (legacy) : créer une liste de favoris manuellement

```bash
# 1. Exporter en texte pour voir le format
python3 recalbox_favorites.py --v9 ~/roms export-text exemple.txt

# 2. Éditer le fichier, ajouter/retirer des jeux

# 3. Simuler d'abord
python3 recalbox_favorites.py --v9 -n ~/roms mark ma-liste.txt

# 4. Appliquer
python3 recalbox_favorites.py --v9 ~/roms mark ma-liste.txt
```

### Correspondance approximative (noms légèrement différents)

```bash
# 90% de similarité : "Sonic the Hedgehog" ≈ "Sonic The Hedgehog (Europe)"
python3 recalbox_favorites.py --threshold 90 ~/roms mark favoris.txt
```

---

## Structure attendue de l'arborescence

```
roms/
├── megadrive/
│   ├── gamelist.xml              ← métadonnées scraper (jamais modifié en v10)
│   ├── gamelist-userdata.ini     ← favoris/stats v10 (créé automatiquement)
│   ├── sonic.md
│   └── streets_of_rage_2.md
├── mame/
│   ├── gamelist.xml
│   ├── 3wonders.zip
│   └── dkong.zip
└── ...
```

## Fichiers générés

| Fichier | Description |
|---|---|
| `recalbox_favorites.log` | Log complet de chaque exécution (DEBUG) |
| `favoris.json` | Export JSON des favoris |
| `favoris.txt` | Export texte des favoris, groupé par système |

---

## Annexe — Alias de systèmes acceptés

Les en-têtes de section `[…]` dans les fichiers texte (`mark`, `export-text`) acceptent
le nom de dossier, le nom long, et tous les alias listés ci-dessous. La casse est ignorée.

| Système (dossier) | En-têtes acceptés |
|---|---|
| **Nintendo** | |
| `nes` | `[nes]`, `[Nintendo NES]`, `[Famicom]`, `[Nintendo Entertainment System]` |
| `fds` | `[fds]`, `[Famicom Disk System]` |
| `snes` | `[snes]`, `[Super Nintendo]`, `[Super NES]`, `[Super Famicom]`, `[Super Nintendo Entertainment System]` |
| `n64` | `[n64]`, `[Nintendo 64]`, `[Nintendo64]` |
| `gb` | `[gb]`, `[Game Boy]`, `[Gameboy]` |
| `gbc` | `[gbc]`, `[Game Boy Color]`, `[Gameboy Color]` |
| `gba` | `[gba]`, `[Game Boy Advance]`, `[Gameboy Advance]` |
| `nds` | `[nds]`, `[Nintendo DS]` |
| `virtualboy` | `[virtualboy]`, `[Virtual Boy]` |
| `pokemini` | `[pokemini]`, `[Pokemon Mini]` |
| **Sega** | |
| `mastersystem` | `[mastersystem]`, `[Sega Master System]`, `[SMS]`, `[Mark III]` |
| `megadrive` | `[megadrive]`, `[Sega Mega Drive]`, `[Mega Drive]`, `[Genesis]`, `[Sega Genesis]` |
| `segacd` | `[segacd]`, `[Sega CD]`, `[Mega CD]`, `[Mega-CD]` |
| `sega32x` | `[sega32x]`, `[Sega 32X]` |
| `saturn` | `[saturn]`, `[Sega Saturn]` |
| `dreamcast` | `[dreamcast]`, `[Sega Dreamcast]` |
| `gamegear` | `[gamegear]`, `[Sega Game Gear]` |
| `sg1000` | `[sg1000]`, `[Sega SG-1000]` |
| `naomi` | `[naomi]`, `[Sega NAOMI]` |
| `atomiswave` | `[atomiswave]`, `[Atomiswave]` |
| **Sony** | |
| `psx` | `[psx]`, `[PlayStation]`, `[PS1]`, `[PSOne]`, `[Playstation 1]` |
| `ps2` | `[ps2]`, `[PlayStation 2]` |
| `psp` | `[psp]`, `[PlayStation Portable]` |
| **NEC** | |
| `pcengine` | `[pcengine]`, `[PC Engine]`, `[TurboGrafx]`, `[TurboGrafx-16]`, `[TG16]` |
| `pcenginecd` | `[pcenginecd]`, `[PC Engine CD]`, `[Turbo CD]` |
| `pc88` | `[pc88]`, `[NEC PC-88]` |
| `pc98` | `[pc98]`, `[NEC PC-98]` |
| **SNK** | |
| `neogeo` | `[neogeo]`, `[Neo Geo]`, `[Neo-Geo]`, `[Neo Geo AES]`, `[Neo Geo MVS]` |
| `neogeocd` | `[neogeocd]`, `[Neo Geo CD]` |
| `ngp` | `[ngp]`, `[Neo Geo Pocket]` |
| `ngpc` | `[ngpc]`, `[Neo Geo Pocket Color]` |
| **Atari** | |
| `atari2600` | `[atari2600]`, `[Atari 2600]`, `[Atari]` |
| `atari5200` | `[atari5200]`, `[Atari 5200]` |
| `atari7800` | `[atari7800]`, `[Atari 7800]` |
| `atarist` | `[atarist]`, `[Atari ST]` |
| `jaguar` | `[jaguar]`, `[Atari Jaguar]` |
| `lynx` | `[lynx]`, `[Atari Lynx]` |
| **Amstrad / Sinclair / Commodore** | |
| `amstradcpc` | `[amstradcpc]`, `[Amstrad CPC]`, `[Amstrad]`, `[CPC]` |
| `zxspectrum` | `[zxspectrum]`, `[ZX Spectrum]`, `[Spectrum]`, `[Sinclair Spectrum]` |
| `c64` | `[c64]`, `[Commodore 64]`, `[Commodore64]` |
| `amiga` | `[amiga]`, `[Amiga]`, `[Commodore Amiga]` |
| `amigacd32` | `[amigacd32]`, `[Amiga CD32]` |
| `vic20` | `[vic20]`, `[Commodore VIC-20]` |
| **Arcade** | |
| `mame` | `[mame]`, `[MAME]` |
| `fba` | `[fba]`, `[FinalBurn Alpha]`, `[Finalburn Alpha]` |
| `fbneo` | `[fbneo]`, `[FinalBurn Neo]`, `[Finalburn Neo]`, `[FB Neo]` |
| **Ordinateurs / Divers** | |
| `dos` | `[dos]`, `[DOS]`, `[MS-DOS]`, `[MSDOS]` |
| `scummvm` | `[scummvm]`, `[ScummVM]`, `[Scumm]` |
| `msx` | `[msx]`, `[MSX]` |
| `msx2` | `[msx2]`, `[MSX2]`, `[MSX 2]` |
| `x68000` | `[x68000]`, `[Sharp X68000]` |
| `x1` | `[x1]`, `[Sharp X1]` |
| `colecovision` | `[colecovision]`, `[ColecoVision]` |
| `intellivision` | `[intellivision]`, `[Intellivision]` |
| `vectrex` | `[vectrex]`, `[Vectrex]` |
| `o2em` | `[o2em]`, `[Magnavox Odyssey 2]` |
| `gw` | `[gw]`, `[Game & Watch]` |
