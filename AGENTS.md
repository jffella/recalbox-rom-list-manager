# recalbox_favorites — AGENTS

Contexte pour les agents IA (Claude, Copilot, etc.) travaillant sur ce projet.

## Fichier principal

`/home/jfella/recalbox_favorites.py` — script Python 3.9+, stdlib uniquement, un seul fichier.

## Objectif

Gérer les favoris EmulationStation / Recalbox via les fichiers `gamelist.xml` et
`gamelist-userdata.ini` répartis dans une arborescence de ROMs.

## Modèle de données

### gamelist.xml (Recalbox v9 / EmulationStation)
```xml
<gameList>
  <game>
    <path>./rom.zip</path>
    <name>Nom du jeu</name>
    <favorite>1</favorite>
  </game>
</gameList>
```

### gamelist-userdata.ini (Recalbox v10)
```
galaga.zip:favorite=true
rtype.zip:timeplayed=138,lastplayed=20260608T191434,playcount=1,favorite=true
mame0274/xybots.zip:favorite=true
```

## Règles importantes

### Ne jamais faire
- Utiliser `xml_path.parent.name` pour le nom du système → vaut `"roms"` dans Recalbox.
  Toujours utiliser `xml_path.relative_to(source).parts[0]`.
- Modifier `gamelist.xml` en mode v10 → seul `gamelist-userdata.ini` est écrit.
- Bootstrapper tous les systèmes en avance → créer `gamelist-userdata.ini` uniquement
  pour les systèmes où un favori est effectivement appliqué (`dirty`).

### Format v10

| Aspect | v9 (`gamelist.xml`) | v10 (`gamelist-userdata.ini`) |
|---|---|---|
| Valeur `<favorite>` mark | `"1"` dans `<favorite>` | `favorite=true` dans l'INI |
| Valeur `<favorite>` unmark | `"0"` dans `<favorite>` | suppression de la clé/fichier |
| Fichier modifié | `gamelist.xml` | `gamelist-userdata.ini` |

### Lecture du champ `<favorite>`

Utiliser `_is_favorite(game)` qui accepte `"1"` et `"true"` (insensible à la casse).
Ne pas comparer directement `get_field(game, "favorite") == "1"`.

### Tolérance XML

`parse_gamelist()` corrige les `&` nus (ex. `"Dale Coop & Raftronaut"`) via `_BARE_AMP_RE`
avant de confier le texte à `ET.fromstring`. Ne pas contourner cette pré-correction.

### Dry-run

`dry_run=True` → aucun fichier écrit, supprimé ni créé.
Toute la logique de recherche/modification en mémoire s'exécute quand même (rapport complet).

### Cache de parsing

`_ensure_loaded(xml_path, trees)` stocke `None` en cas d'échec pour éviter les logs répétés.
Ne jamais appeler `parse_gamelist` directement dans une boucle — passer par `_ensure_loaded`.

## Flux d'appels (mark/apply)

```
mark_from_text
  └─ _parse_text_favorites   ← construit entries[] avec gamelist hint + match_rom + restrict
       └─ system_index : rglob gamelist.xml → _preferred_gamelist
  └─ apply_favorites (via JSON tmp)
       └─ _collect_gamelists → find_gamelists → _preferred_gamelist
       └─ _locate_game        ← recherche path exact → flou → ROM-stem
       └─ set_field favorite  ← "true" si v10, "1" si v9
       └─ (dirty) bootstrap + write_gamelist
```

## CLI

```
recalbox_favorites SOURCE [options] SOUS-COMMANDE [args]

Options globales :
  --threshold PCT   Seuil de similarité 0-100 (défaut 100 = exact)
  -n / --dry-run    Simulation sans écriture
  -v / --verbose    DEBUG sur console
  --v9              Force gamelist.xml (mode Recalbox v9 legacy)

Sous-commandes :
  export OUTPUT_JSON
  export-text OUTPUT_TXT
  apply FAVORITES_JSON
  mark [--by-rom] TEXT_FILE
  unmark
```
