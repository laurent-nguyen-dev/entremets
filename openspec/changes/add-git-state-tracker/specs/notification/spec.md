# Spec Delta

## Purpose

Prévenir l'utilisateur par mail, sans attendre qu'il consulte l'historique, lorsqu'un objet qu'il a jugé critique change d'état sur une machine du cluster.

## ADDED Requirements

### Requirement: Mail récapitulatif par scan
À la fin d'un scan, si au moins un objet de criticité `critical` a changé et que ces changements ont été poussés, l'outil MUST envoyer un seul mail récapitulatif aux destinataires de `notification.recipients`. Les changements d'objets `normal` MUST NOT déclencher de mail.

#### Scenario: Deux objets critiques et un normal
- **WHEN** un scan enregistre le changement de deux objets `critical` et d'un objet `normal`
- **THEN** un seul mail est envoyé, et il ne mentionne que les deux objets critiques

#### Scenario: Seulement des objets normaux
- **WHEN** seuls des objets `normal` ont changé
- **THEN** aucun mail n'est envoyé

### Requirement: Contenu du mail
Le sujet du mail MUST être `[entremets] <cluster>/<contexte>: <n> critical change(s)`. Le corps MUST indiquer le nœud, et pour chaque objet critique modifié, son nom, son type, l'ancien et le nouvel état (ou leur différence pour les états de plusieurs lignes) et l'identifiant du commit.

#### Scenario: Mail après une mise à jour du noyau
- **WHEN** l'objet critique `kernel` change sur `cn0042` du cluster `cluster-x`, contexte `cpu`
- **THEN** le sujet est `[entremets] cluster-x/cpu: 1 critical change(s)` et le corps montre l'ancienne et la nouvelle version du noyau, le nœud `cn0042` et le commit

### Requirement: Pas de mail pour un changement non poussé
L'outil MUST NOT envoyer de mail pour des changements qui n'ont pas pu être poussés, afin que le scan suivant, qui les constatera de nouveau, ne produise pas de doublon.

#### Scenario: Push en échec
- **WHEN** des objets critiques ont changé mais le push échoue
- **THEN** aucun mail n'est envoyé

### Requirement: Envoi par sendmail
Le mail MUST être envoyé par la commande `sendmail` de la machine (ou la commande indiquée par `notification.sendmail`). Un échec d'envoi MUST être signalé sur la sortie d'erreur sans annuler les commits déjà poussés, et le code de sortie du scan reste celui des changements (1).

#### Scenario: sendmail absent sur un nœud de calcul
- **WHEN** un objet critique change sur un nœud où `sendmail` n'existe pas
- **THEN** le changement est commité et poussé, un avertissement indique que le mail n'a pas pu partir, et le scan se termine avec le code 1

#### Scenario: Aucun destinataire
- **WHEN** `notification.recipients` est absent ou vide
- **THEN** aucun mail n'est envoyé et aucun avertissement n'est affiché
