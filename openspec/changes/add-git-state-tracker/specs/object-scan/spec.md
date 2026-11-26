# Spec Delta

## Purpose

Relever l'état des objets sur la machine courante et n'enregistrer dans le dépôt que ce qui a changé, avec un commit explicite par objet, pour que l'historique Git raconte l'évolution du système.

## ADDED Requirements

### Requirement: Scan de la machine courante
`entremets scan --repo <dépôt>` MUST relever, sur la machine où il s'exécute uniquement, l'état de chaque objet applicable au contexte courant. L'outil MUST NOT planifier ses propres exécutions : cron et jobs sont externes.

#### Scenario: Scan dans un job
- **WHEN** un script sbatch exécute `entremets scan --repo R` sur le nœud `gpu017`
- **THEN** les objets sont relevés sur `gpu017` et comparés à la branche `state/gpu`

### Requirement: Filtre par contexte
Un objet qui déclare `contexts` MUST être relevé uniquement dans les contextes listés. Un objet sans `contexts` MUST être relevé dans tous les contextes.

#### Scenario: Objet limité à la partition gpu
- **WHEN** l'objet `nvidia-driver` a `contexts = ["gpu"]` et que le scan a lieu dans le contexte `cpu`
- **THEN** l'objet n'est pas relevé et n'a pas de fichier d'état sur `state/cpu`

### Requirement: Scans en parallèle
Les objets MUST être relevés en parallèle, avec au plus `jobs` relevés simultanés (`defaults.jobs` ou l'option `--jobs`).

#### Scenario: Durée d'un scan
- **WHEN** 8 objets dont la commande dure chacune 1 seconde sont relevés avec `--jobs 8`
- **THEN** le scan se termine en nettement moins de 8 secondes

### Requirement: Délai maximal
Le relevé d'un objet MUST être interrompu s'il dépasse son délai maximal. Un dépassement MUST être signalé sur la sortie d'erreur et MUST NOT modifier l'état stocké de l'objet ni créer de commit.

#### Scenario: Commande bloquée
- **WHEN** la commande d'un objet ne répond pas avant son `timeout`
- **THEN** elle est arrêtée, un avertissement nomme l'objet, et son fichier d'état reste inchangé

### Requirement: Échec enregistré comme état
Si la commande d'un objet se termine avec un code non nul, l'état de l'objet MUST devenir une erreur contenant le code retour et la sortie d'erreur de la commande. Cet état est comparé et commité comme tout autre état.

#### Scenario: Module disparu
- **WHEN** `module show gcc/12.2.0` échoue parce que le module a été retiré
- **THEN** l'état de l'objet devient une erreur, et un commit enregistre ce changement

#### Scenario: Retour à la normale
- **WHEN** un objet en erreur se remet à fonctionner
- **THEN** son nouvel état remplace l'erreur et un commit enregistre ce changement

### Requirement: Rien d'écrit si rien ne change
Si l'état relevé d'un objet est identique à son état stocké sur la branche du contexte, l'outil MUST NOT modifier son fichier ni créer de commit, quel que soit le nœud du contexte sur lequel le scan a lieu.

#### Scenario: Scan sans changement sur un autre nœud
- **WHEN** un scan sur `gpu002` relève les mêmes états que ceux enregistrés depuis `gpu017`
- **THEN** aucun commit n'est créé et rien n'est poussé

### Requirement: Un commit par objet modifié
Chaque objet dont l'état a changé MUST donner un commit séparé sur la branche `state/<contexte>`, qui ne modifie que le fichier d'état de cet objet. Le message MUST avoir pour titre `<objet>: <résumé>` et se terminer par les lignes `Node: <nom d'hôte court>`, `Context: <contexte>` et `Plugin: <type>`.

#### Scenario: Changement de version de paquet
- **WHEN** l'objet `openmpi` passe de `4.1.1-5` à `4.1.5-2` lors d'un scan sur `gpu017`
- **THEN** un commit intitulé `openmpi: 4.1.1-5 -> 4.1.5-2` est créé, avec `Node: gpu017`, `Context: gpu` et `Plugin: package` en fin de message

#### Scenario: Trois objets modifiés
- **WHEN** trois objets changent lors d'un même scan
- **THEN** trois commits distincts sont créés, puis poussés ensemble

#### Scenario: Retrouver les constats d'un nœud
- **WHEN** l'utilisateur lance `git log state/gpu --grep "Node: gpu017"`
- **THEN** il obtient tous les changements constatés sur ce nœud

### Requirement: Résumé du titre de commit
Le résumé du titre MUST être `<ancien> -> <nouveau>` lorsque l'ancien et le nouvel état tiennent chacun sur une ligne courte (80 caractères au plus), `initial state` pour la première apparition d'un objet, `error (exit <code>)` pour un passage en erreur, et `changed` dans les autres cas.

#### Scenario: Fichier texte modifié
- **WHEN** le contenu d'un objet `text-file` de plusieurs lignes change
- **THEN** le titre du commit est `<objet>: changed`, et `git show` montre les lignes modifiées

### Requirement: Suppression des objets retirés
Lors d'un scan, tout fichier d'état de la branche du contexte qui ne correspond plus à un objet applicable à ce contexte MUST être supprimé, avec un commit intitulé `<objet>: removed from configuration`.

#### Scenario: Objet retiré de la configuration
- **WHEN** l'objet `openmpi` a été retiré de `entremets.toml` et qu'un scan a lieu dans le contexte `gpu`
- **THEN** le fichier d'état `openmpi` est supprimé de `state/gpu` par un commit dédié, et son historique reste accessible avec `git log`

### Requirement: Sortie et codes de sortie
Le scan MUST afficher sur la sortie standard une ligne par commit créé, et rien si aucun objet n'a changé. Il MUST se terminer avec le code 0 si aucun changement n'a été enregistré, 1 si des changements ont été enregistrés et poussés, et 2 en cas d'erreur.

#### Scenario: Scan cron sans changement
- **WHEN** un scan lancé par cron ne constate aucun changement
- **THEN** il n'écrit rien sur la sortie standard et se termine avec le code 0

#### Scenario: Scan avec changements
- **WHEN** un scan enregistre et pousse deux changements
- **THEN** il affiche deux lignes et se termine avec le code 1
