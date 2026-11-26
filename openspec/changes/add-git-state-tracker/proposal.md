# Proposal

## Why

Sur les clusters HPC, les administrateurs mettent régulièrement à jour le noyau, l'OS, les pilotes, les paquets et les modules, sans que l'utilisateur sache précisément quoi, quand ni sur quels nœuds. Quand un code se met à planter ou à donner des résultats différents, il est impossible de relier ce comportement à un changement précis de l'environnement. Il faut un historique fiable, daté et explorable de l'état des composants dont dépend le code.

## What Changes

- Création d'`entremets`, un outil en ligne de commande (Rust, binaire statique, sans droits root) qui s'appuie sur **un dépôt Git servant à la fois de configuration et de mémoire**.
- Le dépôt est toujours passé explicitement en entrée (URL ou dépôt nu). L'outil travaille dans un clone temporaire, commite et pousse lui-même.
- La branche `main` contient la configuration (`entremets.toml`) et les plugins du dépôt (`plugins/`). Chaque contexte (nœud de connexion, partition) a sa branche orpheline `state/<contexte>`, avec un fichier d'état par objet.
- Les contextes sont déclarés avec des ensembles de nœuds au format ClusterShell. L'outil déduit son contexte du nom d'hôte.
- Les objets surveillés sont décrits par un type (plugin), des paramètres, une criticité, un délai maximal et un filtre de contextes optionnel.
- Système de plugins, tous dérivés du type de base `command` : plugins **déclaratifs** (commande avec paramètres, éventuellement plusieurs variantes conditionnelles) ou **exécutables** (script versionné dans le dépôt). Plugins fournis : `command`, `kernel`, `os`, `package`, `text-file`, `binary-file`, `env`, `module`.
- `entremets scan` : relève en parallèle les objets du contexte courant. Un objet modifié donne **un commit par objet**, avec le nœud et le contexte dans le message. Un objet inchangé ne produit rien. Le push est automatique.
- Mail récapitulatif par scan pour les objets de criticité `critical`.
- Commandes de gestion : `init`, `check`, `plugins`, `objects`, `add`, `remove`.
- Langue : l'outil, sa configuration et sa documentation sont en anglais. Les documents OpenSpec restent en français pendant la phase de test et seront traduits avant publication.
- Préparation à la publication open source sous double licence MIT / CeCILL-B.
- Hors périmètre de cette version : planification intégrée (cron et jobs restent externes), Spack, contrôle d'homogénéité d'une partition, comparaison entre clusters.

## Capabilities

### New Capabilities

- `state-repository` : dépôt Git en entrée, clone temporaire, organisation des branches (`main` et `state/<contexte>`), création d'un dépôt (`init`), push et reprise en cas de conflit.
- `configuration` : format de `entremets.toml` (contextes, objets, valeurs par défaut, notification), détermination du contexte courant et validation (`check`).
- `plugins` : format des plugins déclaratifs et exécutables, variantes conditionnelles, substitution des paramètres, priorité des plugins du dépôt sur les plugins fournis, plugins fournis en v1 et commande `plugins`.
- `object-scan` : déroulement d'un scan (sélection des objets, parallélisme, délais, échecs, écriture des états, commits par objet, suppression des objets retirés, codes de sortie).
- `object-management` : listage, ajout et retrait d'objets en ligne de commande, avec la trace de leur configuration dans l'historique.
- `notification` : mail récapitulatif des changements d'objets critiques.

### Modified Capabilities

Aucune (projet nouveau).

## Impact

- Nouveau projet Rust à la racine du dépôt (`Cargo.toml`, `src/`, `tests/`), fichiers de licence, README en anglais et CI.
- Dépend, à l'exécution sur le cluster, de la commande `git` et, pour les mails, de `sendmail`.
- N'écrit sur la machine scannée que dans un répertoire temporaire. Ne modifie jamais le système.
- Toute personne pouvant pousser sur le dépôt de configuration peut faire exécuter des commandes sous le compte de l'utilisateur sur le cluster : ce point devra figurer dans la documentation.
