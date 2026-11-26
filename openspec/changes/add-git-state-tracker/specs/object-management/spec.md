# Spec Delta

## Purpose

Permettre de consulter, d'ajouter et de retirer les objets surveillés en ligne de commande, sans éditer `entremets.toml` à la main, tout en gardant dans l'historique Git la trace complète de chaque objet ajouté ou retiré.

## ADDED Requirements

### Requirement: Liste des objets
`entremets objects --repo <dépôt>` MUST lister les objets configurés avec, pour chacun, son nom, son type, ses paramètres, sa criticité, son délai et ses contextes (ou « all »).

#### Scenario: Liste
- **WHEN** la configuration contient trois objets
- **THEN** les trois sont affichés avec leurs attributs, dans l'ordre alphabétique de leur nom

### Requirement: Ajout d'un objet
`entremets add --repo <dépôt> <nom> --type <plugin> [--param <clé>=<valeur>]… [--criticality <niveau>] [--timeout <durée>] [--contexts <c1>,<c2>]` MUST ajouter l'objet à `entremets.toml` sur `main`, puis commiter et pousser. Un nom déjà utilisé MUST être refusé.

#### Scenario: Ajout réussi
- **WHEN** l'utilisateur lance `entremets add --repo R openmpi --type package --param name=openmpi --criticality critical --contexts gpu,cpu`
- **THEN** le bloc `[objects.openmpi]` est ajouté, commité et poussé sur `main`

#### Scenario: Nom déjà utilisé
- **WHEN** l'objet `openmpi` existe déjà
- **THEN** la commande refuse, ne modifie rien et se termine avec le code 2

### Requirement: Validation avant écriture
`add` MUST valider l'objet avec les règles de `check` (plugin connu, paramètres requis présents, paramètres inconnus refusés, contextes existants, durée valide) et MUST NOT écrire quoi que ce soit si l'objet est invalide.

#### Scenario: Paramètre requis manquant
- **WHEN** l'utilisateur ajoute un objet `text-file` sans `--param path=…`
- **THEN** la commande indique le paramètre manquant, ne crée aucun commit et se termine avec le code 2

### Requirement: Retrait d'un objet
`entremets remove --repo <dépôt> <nom>` MUST retirer le bloc de l'objet de `entremets.toml` sur `main`, puis commiter et pousser. Un nom inconnu MUST être refusé avec le code 2.

#### Scenario: Retrait réussi
- **WHEN** l'utilisateur lance `entremets remove --repo R openmpi`
- **THEN** le bloc `[objects.openmpi]` est retiré, commité et poussé, et le prochain scan de chaque contexte supprime le fichier d'état correspondant

### Requirement: Trace de la configuration dans les commits
Le commit créé par `add` MUST être intitulé `Add object <nom>` et celui créé par `remove` `Remove object <nom>`. Le corps de chaque message MUST contenir le bloc TOML complet de l'objet ajouté ou retiré.

#### Scenario: Retrouver un objet retiré
- **WHEN** l'objet `openmpi` a été retiré il y a plusieurs mois
- **THEN** `git log main --grep "Remove object openmpi"` affiche le commit, dont le message contient toute la configuration de l'objet

### Requirement: Mise en forme préservée
`add` et `remove` MUST ne modifier que le bloc de l'objet concerné, en préservant les commentaires, l'ordre et la mise en forme du reste de `entremets.toml`.

#### Scenario: Commentaires conservés
- **WHEN** `entremets.toml` contient des commentaires et qu'un objet est ajouté
- **THEN** le diff du commit ne montre que les lignes du nouvel objet
