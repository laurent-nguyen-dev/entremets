# Spec Delta

## Purpose

Définir comment l'état d'un objet est relevé : chaque type d'objet est un plugin qui dérive du type de base « lancer une commande », écrit soit de façon déclarative, soit comme un script, fourni avec l'outil ou ajouté dans le dépôt.

## ADDED Requirements

### Requirement: Type de base command
Le plugin `command` MUST exécuter la commande donnée par son paramètre `command` avec `/bin/sh -c` et utiliser sa sortie standard comme état de l'objet. Tout autre plugin MUST se ramener à l'exécution d'une commande ou d'un script, avec les mêmes règles de délai, de code retour et de capture.

#### Scenario: Objet de type command
- **WHEN** un objet `[objects.gcc-version]` a `type = "command"` et `command = "gcc -dumpfullversion"`
- **THEN** son état est la sortie standard de `gcc -dumpfullversion`

### Requirement: Plugin déclaratif
Un plugin déclaratif MUST être décrit par un fichier TOML qui déclare une `description`, ses paramètres (nom, caractère obligatoire, valeur par défaut, description) et soit une `command`, soit un `exec`, soit une liste de `variants`. Dans `command`, chaque `{param}` MUST être remplacé par la valeur du paramètre.

#### Scenario: Plugin avec un paramètre
- **WHEN** un plugin déclare `params.name` obligatoire et `command = "rpm -q {name}"`, et qu'un objet de ce type a `name = "openmpi"`
- **THEN** la commande exécutée est `rpm -q 'openmpi'`

### Requirement: Paramètres protégés pour le shell
Lors de la substitution dans une commande, la valeur de chaque paramètre MUST être protégée pour le shell, de sorte qu'elle soit toujours transmise comme un seul argument littéral.

#### Scenario: Valeur avec espace et caractères spéciaux
- **WHEN** un paramètre vaut `my file; rm -rf ~`
- **THEN** la commande reçoit exactement cette chaîne comme un seul argument, et aucune autre commande n'est exécutée

### Requirement: Variantes conditionnelles
Un plugin MAY déclarer plusieurs `variants`, chacune avec une condition `when` (une commande shell) et une `command` ou un `exec`. L'outil MUST utiliser la première variante dont la condition se termine avec le code 0. Si aucune ne convient, l'objet MUST être enregistré en erreur avec la cause « no applicable variant ».

#### Scenario: Machine Debian
- **WHEN** le plugin `package` a une variante RPM (`when = "command -v rpm"`) puis une variante dpkg, sur une machine où seul `dpkg-query` existe
- **THEN** la variante dpkg est utilisée

#### Scenario: Aucune variante applicable
- **WHEN** aucune condition `when` ne réussit
- **THEN** l'état de l'objet est une erreur indiquant qu'aucune variante n'est applicable

### Requirement: Plugin exécutable
Un plugin MAY déclarer `exec = "<script>"`, chemin relatif au dossier `plugins/`. L'outil MUST exécuter ce script en lui passant chaque paramètre comme argument `--<param>=<valeur>` et comme variable d'environnement `ENTREMETS_PARAM_<PARAM>`, et utiliser sa sortie standard comme état.

#### Scenario: Script de plugin
- **WHEN** un objet utilise un plugin avec `exec = "package.sh"` et `name = "openmpi"`
- **THEN** `plugins/package.sh --name=openmpi` est exécuté avec `ENTREMETS_PARAM_NAME=openmpi` dans son environnement, et sa sortie standard devient l'état

### Requirement: Flux capturé
Un plugin MAY déclarer `capture` parmi `stdout` (par défaut), `stderr` ou `both`, pour les commandes qui écrivent leur résultat sur la sortie d'erreur. L'outil MUST utiliser le flux indiqué comme état de l'objet.

#### Scenario: Module écrivant sur stderr
- **WHEN** le plugin `module` déclare `capture = "stderr"`
- **THEN** l'état de l'objet est la sortie d'erreur de la commande

### Requirement: Plugins du dépôt prioritaires
L'outil MUST charger d'abord les plugins fournis avec l'outil, puis les fichiers `plugins/<nom>.toml` du dépôt. Un plugin du dépôt portant le nom d'un plugin fourni MUST le remplacer.

#### Scenario: Plugin fourni remplacé
- **WHEN** le dépôt contient `plugins/package.toml`
- **THEN** ce plugin est utilisé à la place du plugin `package` fourni, et `entremets plugins` l'indique comme provenant du dépôt

### Requirement: Plugins fournis en v1
L'outil MUST fournir les plugins `command`, `kernel` (version du noyau), `os` (nom et version de la distribution), `package` (version d'un paquet, avec détection de RPM, dpkg ou pacman, ou `not installed`), `text-file` (contenu d'un fichier texte), `binary-file` (empreinte SHA-256 et taille), `env` (valeur d'une variable ou `unset`) et `module` (sortie de `module show`, avec Lmod ou Environment Modules).

#### Scenario: Paquet absent
- **WHEN** un objet `package` vise un paquet non installé
- **THEN** son état est `not installed`, et ce n'est pas une erreur

#### Scenario: Fichier binaire modifié
- **WHEN** le contenu d'un fichier suivi par `binary-file` change
- **THEN** son état change (nouvelle empreinte), sans que le contenu du fichier soit stocké

#### Scenario: Variable absente
- **WHEN** un objet `env` vise une variable non définie dans l'environnement du scan
- **THEN** son état est `unset`

### Requirement: Consultation des plugins
La commande `entremets plugins --repo <dépôt>` MUST lister les plugins disponibles avec leur description et leur provenance (fournis ou dépôt). `entremets plugins --repo <dépôt> <nom>` MUST afficher le détail d'un plugin : paramètres obligatoires et optionnels, valeurs par défaut et un exemple d'objet.

#### Scenario: Détail d'un plugin
- **WHEN** l'utilisateur lance `entremets plugins --repo R text-file`
- **THEN** l'outil affiche le paramètre obligatoire `path` et un exemple de bloc `[objects.…]` prêt à copier
