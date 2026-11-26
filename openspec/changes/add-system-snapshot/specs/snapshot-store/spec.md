# Spec Delta

## Purpose

Conserver l'historique des instantanés de chaque hôte dans le répertoire de l'utilisateur, et permettre de le consulter et d'y désigner un instantané précis.

## ADDED Requirements

### Requirement: Enregistrement de chaque relevé
Chaque exécution réussie de `entremets scan` SHALL enregistrer un nouvel instantané, même s'il est identique au précédent, afin de dater précisément la fenêtre où un changement est survenu.

#### Scenario: Deux relevés identiques
- **WHEN** l'utilisateur lance `entremets scan` deux fois sans changement du système entre les deux
- **THEN** deux instantanés distincts sont enregistrés, chacun avec son propre horodatage

### Requirement: Emplacement du stockage
Les instantanés SHALL être stockés dans `$ENTREMETS_DIR` si cette variable est définie, sinon dans `$XDG_DATA_HOME/entremets`, sinon dans `~/.local/share/entremets`. L'option `--store <chemin>` SHALL primer sur ces valeurs.

#### Scenario: Emplacement par défaut
- **WHEN** ni `--store`, ni `ENTREMETS_DIR`, ni `XDG_DATA_HOME` ne sont définis
- **THEN** l'instantané est écrit sous `~/.local/share/entremets`

#### Scenario: Emplacement forcé en ligne de commande
- **WHEN** l'utilisateur lance `entremets scan --store /scratch/moi/entremets` alors que `ENTREMETS_DIR` est défini
- **THEN** l'instantané est écrit sous `/scratch/moi/entremets`

### Requirement: Séparation par hôte
Les instantanés SHALL être rangés par nom d'hôte, afin qu'un répertoire personnel partagé entre plusieurs nœuds ne mélange pas leurs historiques.

#### Scenario: Home partagé entre deux nœuds de connexion
- **WHEN** l'utilisateur lance `entremets scan` sur `login1` puis sur `login2`, avec le même répertoire de stockage
- **THEN** les instantanés de chaque nœud sont rangés séparément et `entremets list` sur `login1` ne montre que ceux de `login1`

### Requirement: Identifiant d'instantané
Chaque instantané SHALL avoir un identifiant unique par hôte, dérivé de son horodatage UTC (format `AAAAMMJJTHHMMSSZ`), triable chronologiquement.

#### Scenario: Identifiant affiché après un relevé
- **WHEN** `entremets scan` se termine avec succès
- **THEN** l'identifiant de l'instantané créé est affiché (sauf en mode `--quiet`)

### Requirement: Listage de l'historique
La commande `entremets list` SHALL afficher les instantanés de l'hôte courant, du plus ancien au plus récent, avec pour chacun son identifiant, la version du noyau et le nombre de différences avec l'instantané précédent. L'option `--host <nom>` SHALL permettre de lister ceux d'un autre hôte.

#### Scenario: Historique de trois relevés
- **WHEN** trois instantanés existent pour l'hôte courant et que l'utilisateur lance `entremets list`
- **THEN** les trois sont affichés dans l'ordre chronologique, avec le nombre de différences de chacun par rapport au précédent

#### Scenario: Aucun instantané
- **WHEN** aucun instantané n'existe pour l'hôte demandé
- **THEN** la commande l'indique clairement et se termine sans erreur

### Requirement: Format d'instantané versionné
Chaque instantané SHALL être un fichier JSON lisible portant un numéro de version de format, afin que les versions futures de l'outil puissent relire les anciens instantanés ou refuser explicitement un format inconnu.

#### Scenario: Format plus récent que l'outil
- **WHEN** l'outil lit un instantané dont la version de format est supérieure à celle qu'il connaît
- **THEN** il signale le fichier concerné et le format attendu, et se termine avec le code 2
