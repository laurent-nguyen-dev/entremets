# Spec Delta

## Purpose

Relever, sans droits root, l'état des composants système dont dépend le code de l'utilisateur (noyau, distribution, glibc, pilote GPU, paquets installés) afin d'en garder une trace datée et comparable.

## ADDED Requirements

### Requirement: Relevé des informations de base
La commande `entremets scan` SHALL relever le nom d'hôte, la date et l'heure du relevé (UTC), la version du noyau, le nom et la version de la distribution (d'après `/etc/os-release`) et la version de la glibc.

#### Scenario: Relevé sur un nœud Linux standard
- **WHEN** l'utilisateur lance `entremets scan` sur un serveur Linux
- **THEN** l'instantané produit contient le nom d'hôte, l'horodatage UTC, la version du noyau, l'identifiant et la version de la distribution et la version de la glibc

### Requirement: Relevé du pilote NVIDIA
La commande `entremets scan` SHALL relever la version du pilote NVIDIA lorsqu'un pilote est chargé sur la machine.

#### Scenario: Machine équipée de GPU NVIDIA
- **WHEN** l'utilisateur lance `entremets scan` sur une machine où le pilote NVIDIA est chargé
- **THEN** l'instantané contient la version du pilote NVIDIA

#### Scenario: Machine sans GPU NVIDIA
- **WHEN** l'utilisateur lance `entremets scan` sur une machine sans pilote NVIDIA
- **THEN** l'instantané indique que le pilote NVIDIA est absent, et le relevé réussit

### Requirement: Relevé des paquets installés
La commande `entremets scan` SHALL relever la liste des paquets installés avec, pour chacun, son nom, son architecture et sa version complète, en interrogeant RPM ou dpkg selon ce qui est disponible.

#### Scenario: Distribution à base de RPM
- **WHEN** l'utilisateur lance `entremets scan` sur une machine RHEL, Rocky ou SUSE
- **THEN** l'instantané contient tous les paquets RPM installés avec nom, architecture et version-release (époque comprise si elle existe)

#### Scenario: Distribution à base de dpkg
- **WHEN** l'utilisateur lance `entremets scan` sur une machine Debian ou Ubuntu
- **THEN** l'instantané contient tous les paquets dpkg installés avec nom, architecture et version

### Requirement: Tolérance aux sources indisponibles
Lorsqu'une source d'information est absente ou illisible, le relevé SHALL la marquer comme indisponible dans l'instantané, avec la raison, et poursuivre avec les autres sources.

#### Scenario: Gestionnaire de paquets introuvable
- **WHEN** ni `rpm` ni `dpkg-query` ne sont disponibles sur la machine
- **THEN** l'instantané marque la source « paquets » comme indisponible avec la raison, les autres informations sont relevées et la commande ne se termine pas en erreur

### Requirement: Aucune modification du système
Le relevé MUST fonctionner sans droits root et MUST NOT modifier quoi que ce soit en dehors du répertoire de stockage des instantanés.

#### Scenario: Utilisateur sans privilèges
- **WHEN** un utilisateur non privilégié lance `entremets scan`
- **THEN** le relevé aboutit et seul le répertoire de stockage est modifié
