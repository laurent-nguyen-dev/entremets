# Spec Delta

## Purpose

Décrire, dans un fichier `entremets.toml` lisible et commenté, le cluster, ses contextes, les objets à surveiller et la notification, et garantir que cette configuration est valide avant toute utilisation.

## ADDED Requirements

### Requirement: Fichier de configuration TOML
La configuration MUST être le fichier `entremets.toml` à la racine de la branche `main`, au format TOML. Il MUST contenir le nom du cluster (`cluster`), une table `contexts`, une table `objects`, et MAY contenir une table `defaults` et une table `notification`.

#### Scenario: Configuration minimale
- **WHEN** `entremets.toml` contient `cluster`, un contexte et un objet
- **THEN** la configuration est valide

### Requirement: Contextes déclarés par ensembles de nœuds
Chaque entrée de `contexts` MUST associer un nom de contexte à un ensemble de nœuds au format ClusterShell (par exemple `gpu = "gpu[001-064]"`), avec intervalles, zéros de tête, listes séparées par des virgules et plusieurs groupes entre crochets. Un même nœud MUST NOT appartenir à deux contextes.

#### Scenario: Ensemble avec intervalle
- **WHEN** le contexte `cpu` vaut `"cn[0001-0512]"`
- **THEN** les nœuds `cn0001` à `cn0512` appartiennent au contexte `cpu`, et `cn513` n'y appartient pas

#### Scenario: Contextes qui se chevauchent
- **WHEN** le nœud `gpu001` appartient à la fois aux contextes `gpu` et `visu`
- **THEN** la validation échoue en citant le nœud et les deux contextes

### Requirement: Détermination du contexte courant
L'outil MUST déterminer le contexte courant en cherchant le nom d'hôte court de la machine (avant le premier `.`) dans les ensembles déclarés. L'option `--context <nom>` MUST primer sur cette détection et MUST désigner un contexte déclaré.

#### Scenario: Nœud déclaré
- **WHEN** `entremets scan` s'exécute sur `cn0042.cluster-x.example.org` et que `cpu = "cn[0001-0512]"`
- **THEN** le contexte courant est `cpu`

#### Scenario: Nœud inconnu
- **WHEN** le nom d'hôte n'appartient à aucun contexte et que `--context` n'est pas fourni
- **THEN** l'outil affiche le nom d'hôte et la liste des contextes déclarés, ne scanne rien et se termine avec le code 2

#### Scenario: Contexte forcé inexistant
- **WHEN** l'utilisateur passe `--context visu` alors qu'aucun contexte `visu` n'est déclaré
- **THEN** l'outil le signale et se termine avec le code 2

### Requirement: Description d'un objet
Chaque objet MUST être une table `[objects.<nom>]` avec un `type` (nom d'un plugin) et les paramètres exigés par ce plugin. Il MAY préciser `criticality` (`critical` ou `normal`), `timeout` (durée, par exemple `30s` ou `2m`) et `contexts` (liste de contextes où l'objet s'applique). Un nom d'objet MUST respecter `[a-z0-9][a-z0-9._-]*`.

#### Scenario: Objet complet
- **WHEN** la configuration contient `[objects.nvidia-driver]` avec `type = "command"`, `command = "…"`, `criticality = "critical"`, `timeout = "10s"` et `contexts = ["gpu"]`
- **THEN** l'objet est valide

### Requirement: Valeurs par défaut
La table `defaults` MAY définir `criticality`, `timeout` et `jobs` (nombre maximal de scans simultanés). En leur absence, les valeurs MUST être respectivement `normal`, `30s` et le nombre de processeurs de la machine.

#### Scenario: Objet sans criticité ni délai
- **WHEN** un objet ne précise ni `criticality` ni `timeout`, et que `defaults` n'est pas défini
- **THEN** il est traité comme `normal`, avec un délai maximal de 30 secondes

### Requirement: Validation de la configuration
La commande `entremets check --repo <dépôt>` MUST valider la syntaxe TOML et le sens de la configuration et des plugins du dépôt : type de plugin connu, paramètres requis présents et paramètres inconnus refusés, contextes cités existants, durées valides, ensembles de nœuds valides et disjoints. Chaque erreur MUST indiquer le fichier, la ligne, la colonne et la cause.

#### Scenario: Paramètre manquant
- **WHEN** un objet de type `package` n'a pas de paramètre `name`
- **THEN** `check` affiche une erreur de la forme `entremets.toml:14:1: object "openmpi": plugin "package" requires parameter "name"` et se termine avec le code 2

#### Scenario: Configuration valide
- **WHEN** la configuration et les plugins sont valides
- **THEN** `check` l'indique et se termine avec le code 0

### Requirement: Configuration valide avant tout scan
`entremets scan` MUST valider la configuration avec les mêmes règles que `check` avant de scanner, et MUST NOT scanner quoi que ce soit si elle est invalide.

#### Scenario: Configuration cassée
- **WHEN** quelqu'un a poussé sur `main` une configuration invalide
- **THEN** le scan affiche les erreurs, ne modifie aucune branche d'état et se termine avec le code 2
