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
| `--v10` | Force le format Recalbox v10 — voir section dédiée |

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
(`[Genesis]`) et les noms de dossiers (`[megadrive]`).

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
compteurs de jeu, date de dernière partie) dans un fichier séparé :
`gamelist.recalbox.xml`, placé dans le même répertoire que `gamelist.xml`.

### Détection automatique

Sans aucune option, le script détecte automatiquement la version :
- `gamelist.recalbox.xml` présent → utilisé à la place de `gamelist.xml`
- Absent → comportement v9 classique

### Option `--v10` — Forcer le format v10

```bash
python3 recalbox_favorites.py --v10 ~/roms mark favoris.txt
```

Force l'utilisation de `gamelist.recalbox.xml` pour toutes les opérations.
Si le fichier n'existe pas encore pour un système, il est **créé automatiquement**
par copie du `gamelist.xml` existant, uniquement pour les systèmes qui ont un favori
à appliquer.

```bash
# Vérifier ce qui serait créé sans toucher au disque
python3 recalbox_favorites.py --v10 -n ~/roms mark favoris.txt
```

### `unmark --v10` — Supprimer les fichiers v10

```bash
python3 recalbox_favorites.py --v10 ~/roms unmark
```

En mode v10, `unmark` **supprime** les fichiers `gamelist.recalbox.xml` au lieu
de mettre `<favorite>0</favorite>`. C'est l'opération inverse du bootstrap.

### Format gamelist.recalbox.xml

```xml
<?xml version="1.0"?>
<gameList>
    <game>
        <path>./Resident Evil 3 - Nemesis (France).chd</path>
        <favorite>true</favorite>
        <playcount>12</playcount>
        <lastplayed>20260609T143000</lastplayed>
    </game>
</gameList>
```

Différences avec v9 : `true` au lieu de `1`, indentation 4 espaces,
déclaration XML sans attribut `encoding`, contient uniquement les métadonnées
utilisateur (pas les champs scraper).

---

## Exemples complets

### Workflow v9 : sauvegarde et restauration des favoris

```bash
# 1. Exporter les favoris actuels
python3 recalbox_favorites.py ~/roms export sauvegarde.json

# 2. (après réinstallation) Restaurer
python3 recalbox_favorites.py ~/roms apply sauvegarde.json
```

### Workflow v9 : créer une liste de favoris manuellement

```bash
# 1. Exporter en texte pour voir le format
python3 recalbox_favorites.py ~/roms export-text exemple.txt

# 2. Éditer le fichier, ajouter/retirer des jeux

# 3. Simuler d'abord
python3 recalbox_favorites.py -n ~/roms mark ma-liste.txt

# 4. Appliquer
python3 recalbox_favorites.py ~/roms mark ma-liste.txt
```

### Workflow v10 : initialiser les favoris depuis une liste de ROMs

```bash
# 1. Créer les gamelist.recalbox.xml et marquer les favoris
python3 recalbox_favorites.py --v10 ~/roms mark --by-rom favoris-roms.txt

# 2. Tout effacer (supprime les .recalbox.xml)
python3 recalbox_favorites.py --v10 ~/roms unmark
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
│   ├── gamelist.xml              ← v9
│   ├── gamelist.recalbox.xml     ← v10 (créé par --v10 ou présent nativement)
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
