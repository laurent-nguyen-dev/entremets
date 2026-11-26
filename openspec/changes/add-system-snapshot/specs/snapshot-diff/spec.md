# Spec Delta

## Purpose

Montrer précisément ce qui a changé sur le système entre deux instantanés, pour relier un changement de comportement d'un code à une mise à jour datée.

## ADDED Requirements

### Requirement: Comparaison de deux instantanés
La commande `entremets diff [A] [B]` SHALL comparer l'instantané A (le plus ancien) à l'instantané B. Sans argument, elle SHALL comparer les deux derniers instantanés de l'hôte courant. Avec un seul argument, elle SHALL comparer cet instantané au dernier.

#### Scenario: Sans argument
- **WHEN** l'utilisateur lance `entremets diff` et que l'hôte a au moins deux instantanés
- **THEN** l'avant-dernier et le dernier instantané sont comparés

#### Scenario: Moins de deux instantanés
- **WHEN** l'utilisateur lance `entremets diff` et que l'hôte a moins de deux instantanés
- **THEN** la commande explique qu'il faut au moins deux relevés et se termine avec le code 2

#### Scenario: Identifiant inconnu
- **WHEN** l'utilisateur passe un identifiant qui ne correspond à aucun instantané
- **THEN** la commande le signale et se termine avec le code 2

### Requirement: Désignation abrégée d'un instantané
Un instantané SHALL pouvoir être désigné par un préfixe de son identifiant (par exemple `20261126` pour un jour), à condition que ce préfixe n'en désigne qu'un seul.

#### Scenario: Préfixe ambigu
- **WHEN** l'utilisateur passe un préfixe correspondant à plusieurs instantanés
- **THEN** la commande liste les instantanés correspondants et se termine avec le code 2

### Requirement: Contenu du rapport
Le rapport SHALL indiquer, pour les informations de base (noyau, distribution, glibc, pilote NVIDIA), l'ancienne et la nouvelle valeur de chaque élément modifié. Pour les paquets, il SHALL lister séparément les ajouts, les suppressions et les changements de version (ancienne vers nouvelle). Un paquet est identifié par son nom et son architecture.

#### Scenario: Mise à jour de paquet
- **WHEN** le paquet `openmpi` (x86_64) passe de la version `4.1.1-5` à `4.1.5-2` entre A et B
- **THEN** le rapport le liste parmi les changements de version avec `4.1.1-5 → 4.1.5-2`

#### Scenario: Changement de noyau
- **WHEN** la version du noyau diffère entre A et B
- **THEN** le rapport affiche l'ancienne et la nouvelle version du noyau en tête

#### Scenario: Source devenue indisponible
- **WHEN** une source disponible dans A est indisponible dans B
- **THEN** le rapport le signale explicitement plutôt que de présenter ses éléments comme supprimés

### Requirement: Format de sortie
Le rapport SHALL être affiché par défaut en texte lisible, regroupé par catégorie. L'option `--json` SHALL produire le même contenu en JSON pour un traitement par script.

#### Scenario: Sortie JSON
- **WHEN** l'utilisateur lance `entremets diff --json`
- **THEN** la sortie standard contient uniquement un document JSON valide décrivant les différences

### Requirement: Codes de sortie
Les commandes `diff` et `scan` SHALL se terminer avec le code 0 si aucune différence n'est trouvée, 1 si des différences sont trouvées et 2 en cas d'erreur. Pour `scan`, la comparaison se fait avec l'instantané précédent ; s'il n'y en a pas, le code est 0.

#### Scenario: Aucune différence
- **WHEN** les deux instantanés comparés sont identiques (horodatage mis à part)
- **THEN** la commande indique qu'il n'y a aucun changement et se termine avec le code 0

#### Scenario: Premier relevé
- **WHEN** l'utilisateur lance `entremets scan` pour la première fois sur un hôte
- **THEN** l'instantané est enregistré et la commande se termine avec le code 0

### Requirement: Mode silencieux pour cron
Avec `--quiet`, `entremets scan` SHALL n'écrire rien sur la sortie standard s'il n'y a aucune différence avec l'instantané précédent, et SHALL écrire le rapport de différences sinon. Les erreurs SHALL toujours être écrites sur la sortie d'erreur.

#### Scenario: Relevé cron sans changement
- **WHEN** cron lance `entremets scan --quiet` et que rien n'a changé
- **THEN** rien n'est affiché, l'instantané est tout de même enregistré et le code de sortie est 0

#### Scenario: Relevé cron après une mise à jour
- **WHEN** cron lance `entremets scan --quiet` après une mise à jour de paquets
- **THEN** le rapport de différences est affiché (cron l'envoie alors par e-mail) et le code de sortie est 1
