# Format de monde natif — versions 1 et 2

Un **monde natif** décrit un jeu pour un randomizer multiworld compatible avec Archipelago : ses objets, ses lieux, ses régions, ses règles et ses options, dans un fichier TOML, **sans code**. Raftup le lit, le vérifie et le compile vers son moteur de génération.

Ce dossier est autonome : il pourra devenir son propre dépôt tel quel. Licence : MIT OR Apache-2.0.

- `README.md` : la spécification (ce fichier) ;
- `exemples/apquest/world.toml` : APQuest, réécrit depuis Archipelago 0.6.8 (MIT) ;
- `exemples/minimal/world.toml` : le plus petit monde valide.

Décision et raisons : ADR 0041 du dépôt Raftup.

## Principes

1. **Déclaratif** : un monde ne contient aucun code ; le lire n'exécute rien.
2. **Ordonné** : toutes les listes sont des tableaux TOML (`[[…]]`). L'ordre du fichier est l'ordre de création des éléments, donc la génération est reproductible.
3. **Options résolues à la compilation** : un élément peut n'exister qu'avec certaines options (`when`), une valeur peut en dépendre (liste de cas). Les règles d'accès qui restent ne dépendent que de l'inventaire.
4. **Rien qu'Archipelago ne sache faire** : tout ce que le format exprime a un équivalent dans un apworld, pour qu'un monde natif puisse être exporté en apworld.

## Structure du fichier

```toml
format_version = 1          # obligatoire : 1 ou 2 (voir « Versions »)

[game]                      # obligatoire
[[options]]                 # options du joueur (YAML)
[[items]]                   # sortes d'objets
[[item_groups]]             # groupes d'objets
[[regions]]                 # régions (au moins une)
[[entrances]]               # passages orientés entre régions
[[locations]]               # lieux qui reçoivent un objet
[[location_groups]]         # groupes de lieux
[[events]]                  # lieux d'événement avec leur objet d'événement
[[pool]]                    # objets à placer
[filler]                    # remplissage jusqu'au nombre de lieux
[[start]]                   # inventaire de départ
[[slot_data]]               # données envoyées au client du jeu
```

Un champ inconnu est une erreur (on repère ainsi les fautes de frappe).

### `[game]`

| Champ | Type | Sens |
| --- | --- | --- |
| `name` | texte | Nom du jeu, celui des YAML et du datapackage (obligatoire). |
| `start_region` | texte | Région de départ ; par défaut, la première région présente. |
| `goal` | règle | Condition de victoire (obligatoire). |

### `[[options]]`

| Champ | Type | Sens |
| --- | --- | --- |
| `name` | texte | Nom dans le YAML (`hard_mode`). |
| `display` | texte | Nom affiché (spoiler) ; par défaut `hard_mode` → `Hard Mode`. |
| `type` | `toggle`, `range`, `choice` | Oui/non, nombre borné, choix dans une liste. |
| `default` | booléen, entier ou texte | Valeur par défaut (selon le type). |
| `min`, `max` | entiers | Bornes d'un `range` (comprises). |
| `choices` | liste de textes | Valeurs d'un `choice` ; la n-ième vaut `n - 1`. |
| `aliases` | table texte → texte | Autres noms acceptés dans le YAML (`kitty = "cat"`). |

Un nom d'option ne contient que des lettres ASCII, des chiffres et `_`, et ne reprend pas une option commune à tous les jeux (`progression_balancing`, `accessibility`, `local_items`, `non_local_items`, `start_inventory`, `start_inventory_from_pool`, `start_hints`, `start_location_hints`, `exclude_locations`, `priority_locations`, `item_links`, `plando_items`, `plando_connections`, `plando_texts`).

Valeur d'une option, dans les règles et le `slot_data` : `toggle` → 0 ou 1, `range` → le nombre, `choice` → la position (0, 1, …). Le YAML d'un joueur accepte une valeur, des poids (`{true: 30, false: 70}`) ou `random`, comme Archipelago.

### `[[items]]`

| Champ | Type | Sens |
| --- | --- | --- |
| `name` | texte | Nom unique. |
| `id` | entier > 0 | Numéro réseau (datapackage). |
| `classification` | classification ou cas | `progression`, `useful`, `filler`, `trap`, `skip_balancing`, combinables avec `+` (`progression+useful`). |

Les objets existent toujours (ils sont dans le datapackage) ; c'est le pool qui dit lesquels on place.

### `[[item_groups]]` et `[[location_groups]]`

`name` et `items` (ou `locations`) : liste de noms. Utilisés par `has_group` et écrits dans le datapackage.

### `[[regions]]`

`name`, et `when` facultatif. Chaque région présente doit être atteignable depuis la région de départ en suivant les entrées présentes (règles ignorées) : sinon, c'est une erreur.

### `[[entrances]]`

| Champ | Type | Sens |
| --- | --- | --- |
| `name` | texte | Nom unique. |
| `from`, `to` | textes | Régions de départ et d'arrivée. |
| `rule` | règle | Condition pour passer ; `true` par défaut. |
| `when` | condition | Présente seulement si la condition est vraie. |

Une entrée présente qui touche une région absente est une erreur : écrire le même `when` sur les deux.

### `[[locations]]`

| Champ | Type | Sens |
| --- | --- | --- |
| `name` | texte | Nom unique. |
| `id` | entier > 0 | Numéro réseau. |
| `region` | texte ou cas | Région du lieu. |
| `rule` | règle | Condition d'accès ; `true` par défaut. |
| `progress` | `default`, `priority`, `excluded` | Lieu prioritaire (reçoit de la progression) ou exclu (seulement du remplissage). |
| `when` | condition | Présent seulement si la condition est vraie. |

### `[[events]]`

Un événement est un lieu **sans numéro** où un **objet d'événement** (sans numéro, de progression) est posé d'office : « bouton pressé », « boss vaincu ». Champs : `name`, `region`, `rule`, `when`, et `item` (nom d'un nouvel objet d'événement, distinct des `[[items]]`).

En version 1, chaque événement a son propre objet. En **version 2**, plusieurs événements peuvent donner le **même** objet d'événement : il est créé une fois, chaque événement en pose un exemplaire, et une règle le compte (`has("Pink Plort", 3)`). C'est ce que font beaucoup d'apworlds.

### `[[pool]]`, `[filler]`, `[[start]]`

- `[[pool]]` : `item`, `count` (nombre, 1 par défaut), `when`. Le pool est construit dans l'ordre de la liste.
- `[filler]` : `item` (objet de remplissage), et facultatifs `trap` (objet piège) et `trap_percent` (nombre de 0 à 100). Le pool est complété jusqu'au nombre de lieux : pour chaque place, un tirage de 0 à 99 ; s'il est inférieur à `trap_percent`, c'est un piège. Sans `[filler]`, le pool doit remplir exactement les lieux.
- `[[start]]` : `item`, `count`, `when`. Objets reçus dès le départ (hors pool).

### `[[slot_data]]`

`key`, puis soit `option` (nom d'une option : sa valeur entière), soit `value` (valeur littérale). Dans l'ordre de la liste.

## Nombres

Un champ nombre (`count`, `trap_percent`) accepte un entier, ou un texte `"option(nom)"` qui prend la valeur d'une option `range`.

## Cas

Une valeur qui dépend des options s'écrit comme une liste de cas ; le premier dont la condition est vraie gagne, et le dernier n'a pas de `when` :

```toml
region = [
    { when = "option(hammer)", value = "Top Middle Room" },
    { value = "Overworld" },
]
```

## Mini-langage de règles

### Grammaire

```text
expr       = or
or         = and { "or" and }
and        = unary { "and" unary }
unary      = "not" unary | comparison
comparison = primary [ ("==" | "!=" | "<" | "<=" | ">" | ">=") value ]
primary    = "true" | "false" | "(" expr ")" | call
call       = ident "(" [ arg { "," arg } ] ")"
arg        = name | number | call
name       = ident | string
value      = ident | string | number
ident      = lettre ou "_", puis lettres, chiffres, "_"
string     = '"' … '"'   (échappements \" et \\)
number     = chiffres
```

Les mots `and`, `or`, `not`, `true`, `false` sont réservés : un objet qui s'appelle ainsi s'écrit entre guillemets.

### Fonctions

| Fonction | Sens |
| --- | --- |
| `has(objet)`, `has(objet, n)` | Au moins 1 (ou `n`) exemplaire. `n` peut être `option(nom)`. |
| `has_count(objet, n)` | Comme `has(objet, n)`. |
| `has_all(a, b, …)` | Au moins un de chacun. |
| `has_any(a, b, …)` | Au moins un de l'un d'eux. |
| `has_group(groupe)`, `has_group(groupe, n)` | Au moins 1 (ou `n`) objets du groupe, tous membres confondus. |
| `can_reach(nom)` | La région ou le lieu est atteignable (erreur si le nom désigne les deux). |
| `can_reach_region(nom)`, `can_reach_location(nom)`, `can_reach_entrance(nom)` | Sans ambiguïté. |
| `option(nom)` | Option `toggle` activée (statique). |
| `option(nom) == valeur` | Comparaison d'une option (statique) ; pour un `choice`, la valeur est un nom de choix. |

### Types

- Une expression est **statique** si elle ne dépend que des options (`true`, `false`, `option`, comparaisons) ; sinon elle est **dynamique**.
- `not` ne s'applique qu'à une expression statique : une règle d'accès ne peut que devenir vraie quand l'inventaire grandit.
- Une condition `when` doit être statique.
- Une option `range` ou `choice` ne s'utilise que dans une comparaison ; une option `toggle` peut s'utiliser seule.

### Compilation

Les parties statiques sont calculées avec les options du joueur, puis simplifiées : `x and true` → `x`, `x or true` → `true`, `x and false` → `false`. Un `can_reach` vers une région ou un lieu absent vaut `false`.

### Erreurs

Chaque erreur donne la ligne et la colonne dans le fichier :

```text
ligne 112, colonne 28 : objet « Swrod » inconnu
```

Les positions sont exactes dans une chaîne littérale TOML (`'…'`), recommandée pour les règles qui contiennent des guillemets.

## Limites

Fichier de 4 Mio au plus ; 100 000 éléments par liste ; règle de 16 Kio ; 64 niveaux d'imbrication ; noms de 200 caractères.

## Versions

| Version | Changement |
| --- | --- |
| 1 | Première version. |
| 2 | Plusieurs `[[events]]` peuvent donner le même objet d'événement. |

Un lecteur de la version 2 lit aussi la version 1. Écrire `format_version = 1` quand on n'a pas besoin de la 2 : le fichier reste lisible par plus d'outils.

## Ce que le format ne sait pas encore faire

Règles d'objet (quels objets un lieu accepte, `local_items`), objets à placer tôt, `accessibility`, entrées mélangées, remplissages pondérés, noms spéciaux des options numériques. Ils arriveront avec une `format_version` plus haute.

## Conversion depuis et vers un apworld

Raftup sait convertir un apworld en monde natif, et l'inverse (ADR 0042 du dépôt Raftup, outil `raftup-monde`) :

- `raftup-monde importer export.json --seed AP_xxx.zip` écrit un `world.toml` à partir de l'export d'une génération. Il décrit le monde **pour les options de cette génération** ; les options de l'apworld sont recopiées en commentaire, et la liste « À vérifier » en tête du fichier dit ce que l'outil a dû supposer.
- `raftup-monde apworld world.toml` écrit un `.apworld` pour Archipelago : même datapackage, règles en fonctions Python. Tout monde natif peut ainsi tourner dans Archipelago.
