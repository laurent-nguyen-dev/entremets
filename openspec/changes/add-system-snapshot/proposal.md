# Proposal

## Why

Sur les clusters HPC, les administrateurs mettent régulièrement à jour le noyau, l'OS, la glibc, les pilotes GPU et les paquets système, sans que l'utilisateur sache précisément quoi ni quand. Quand un code se met à planter, à donner des résultats différents ou à ralentir, il est aujourd'hui impossible de relier ce comportement à une mise à jour précise. Il faut un historique daté et comparable de l'état du système vu par l'utilisateur.

## What Changes

- Création d'un outil en ligne de commande `entremets` (Rust, binaire unique, sans droits root).
- `entremets scan` : relève l'état du système (noyau, distribution, glibc, pilote NVIDIA, paquets installés RPM ou DEB) et l'enregistre dans un instantané daté, rangé par nom d'hôte.
- `entremets list` : liste les instantanés enregistrés pour un hôte.
- `entremets diff` : compare deux instantanés (par défaut les deux derniers) et affiche ce qui a été ajouté, retiré ou a changé de version, en texte lisible ou en JSON.
- Mode adapté à cron : `scan --quiet` n'affiche rien si rien n'a changé, et affiche le diff sinon (cron envoie alors un e-mail). Codes de sortie explicites : 0 = pas de changement, 1 = changements détectés, 2 = erreur.
- Hors périmètre de cette première version : bibliothèques liées à un exécutable (ldd), modules Lmod et compilateurs, environnements Python et Conda, scans dans les jobs Slurm, comparaison entre hôtes différents, purge de l'historique.

## Capabilities

### New Capabilities

- `system-snapshot` : relevé des informations système (noyau, OS, glibc, pilote NVIDIA, paquets) et tolérance aux sources indisponibles.
- `snapshot-store` : enregistrement des instantanés, emplacement de stockage, séparation par hôte et listage de l'historique.
- `snapshot-diff` : comparaison de deux instantanés, format du rapport, codes de sortie et mode silencieux pour cron.

### Modified Capabilities

Aucune (projet nouveau).

## Impact

- Nouveau projet Rust (crate `entremets`) à la racine du dépôt.
- Dépendances : `clap` (arguments de ligne de commande), `serde` et `serde_json` (format des instantanés), `anyhow` (gestion des erreurs).
- Écrit uniquement dans le répertoire de l'utilisateur (par défaut `~/.local/share/entremets/`). Ne modifie jamais le système.
- Doit tourner sur des clusters aux glibc anciennes, d'où une compilation statique (musl) prévue (voir design.md).
