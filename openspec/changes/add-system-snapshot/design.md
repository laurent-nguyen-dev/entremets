# Design

## Context

Le dépôt est vide : il s'agit d'un nouveau projet (motivation : voir proposal.md, section Why). L'outil tournera sur des nœuds de connexion de clusters HPC :
- l'utilisateur n'a pas les droits root ;
- la glibc peut être ancienne (RHEL/Rocky 7 à 9 par exemple) ;
- le répertoire personnel est souvent partagé entre plusieurs nœuds, avec des quotas ;
- on ne peut pas toujours installer une chaîne de compilation Rust sur le cluster.

## Goals / Non-Goals

**Goals:**
- Un binaire unique, copiable tel quel sur n'importe quel nœud Linux x86_64.
- Un code organisé pour accueillir facilement de nouvelles sources (ldd, modules, Conda…) dans les versions suivantes.
- Des instantanés lisibles à la main (JSON), pour pouvoir les inspecter ou les retraiter sans l'outil.

**Non-Goals:**
- Pas de démon ni de planificateur intégré : la planification est confiée à cron.
- Pas de base de données.
- Pas d'interface graphique ou web.

## Decisions

### Binaire statique musl
Compilation pour la cible `x86_64-unknown-linux-musl`, ce qui donne un binaire statique indépendant de la glibc du cluster.
- *Alternative* : cible gnu, plus simple, mais un binaire compilé sur une machine récente refuse de démarrer sur une glibc plus ancienne.
- *Conséquence* : la version de la glibc ne peut pas être lue par un appel à `gnu_get_libc_version()`, puisque le binaire n'est pas lié à la glibc. Voir la décision suivante.

### Lecture des informations par fichiers et commandes
Chaque source est lue de la manière la plus robuste possible, sans privilèges :

| Information | Source principale | Repli |
|---|---|---|
| Noyau | `/proc/sys/kernel/osrelease` | `uname -r` |
| Distribution | `/etc/os-release` (`ID`, `VERSION_ID`, `PRETTY_NAME`) | `/usr/lib/os-release` |
| glibc | `getconf GNU_LIBC_VERSION` | première ligne de `ldd --version` |
| Pilote NVIDIA | `/proc/driver/nvidia/version` | `nvidia-smi --query-gpu=driver_version --format=csv,noheader` |
| Paquets RPM | `rpm -qa --queryformat '%{NAME}\t%{ARCH}\t%{EPOCH}:%{VERSION}-%{RELEASE}\n'` | — |
| Paquets dpkg | `dpkg-query -W -f '${Package}\t${Architecture}\t${Version}\n'` | — |

Pour RPM, une époque `(none)` est retirée de la version. Les commandes externes sont lancées avec un délai maximal (30 s), pour qu'un `nvidia-smi` bloqué ne fige pas un cron.

### Architecture en « collecteurs »
Un trait `Collector`, avec un nom et une méthode `collect() -> Result<Value, Unavailable>`. `scan` appelle chaque collecteur et range le résultat sous son nom dans l'instantané : `{"status": "ok", "data": …}` ou `{"status": "unavailable", "reason": "…"}`. Ajouter une source revient à écrire un collecteur. Le diff est générique pour les sources de type « clé → valeur » (les paquets) et dédié pour les informations de base.
- *Alternative* : une structure fixe avec un champ par information. C'est plus simple au départ, mais chaque nouvelle source toucherait alors le format, le scan et le diff.

### Format et stockage
- Fichier `<store>/<hostname>/<id>.json`, où `<id>` vaut par exemple `20261126T072300Z`. Le nom de fichier sert d'index : aucun fichier d'index à maintenir, et `list` se contente de trier les noms.
- Champ `format_version: 1` à la racine.
- Paquets stockés sous la forme d'un objet `"nom.arch": "version"`, trié, qui donne un JSON stable et un diff trivial.
- Écriture atomique (fichier temporaire puis `rename`), pour qu'un cron interrompu ne laisse pas d'instantané tronqué.
- Taille estimée : de 50 à 150 Ko pour 1 500 à 3 000 paquets. Un relevé quotidien représente donc environ 50 Mo par an au pire, ce qui est acceptable pour une v1 (la purge viendra plus tard).
- Nom d'hôte : nom court (avant le premier `.`), lu dans `/proc/sys/kernel/hostname`.

### Comparaison
Deux instantanés sont « identiques » si toutes leurs sources ont le même statut et les mêmes données. L'horodatage est ignoré. Le diff produit une structure intermédiaire (`DiffReport`) sérialisable en JSON, et le rendu texte part de cette même structure, ce qui garantit que `--json` et la sortie texte ont le même contenu.

### Dépendances Rust
`clap` (derive), `serde`, `serde_json` et `anyhow`. Pas de `chrono` : l'horodatage UTC est formaté à la main depuis `SystemTime`, ce qui ne demande qu'une petite fonction de conversion de date testée unitairement. Pour les tests : `tempfile`, et des collecteurs simulés (fixtures de sorties `rpm` et `dpkg` dans `tests/fixtures/`).

## Risks / Trade-offs

- [Nom d'hôte instable ou générique sur certains clusters] → option `--host` sur `scan` pour forcer le nom.
- [Scan lancé sur un nœud de calcul avec un home partagé] → l'historique de ce nœud est simplement rangé à part. Aucun mélange, mais aussi aucune comparaison entre nœuds en v1.
- [Un nœud de connexion n'a pas forcément les mêmes pilotes et paquets que les nœuds de calcul] → limite connue de la v1, notée dans le README. Les scans dans les jobs Slurm viendront plus tard.
- [Liste de paquets longue dans le mail cron] → le rapport texte groupe et trie. Un éventuel mode résumé pourra être ajouté si besoin.
- [La sortie de `rpm` est lente sur certaines machines (plusieurs secondes)] → acceptable pour un usage quotidien. Le délai maximal protège des blocages.

## Migration Plan

Aucune migration nécessaire (première version). Déploiement : `cargo build --release --target x86_64-unknown-linux-musl`, copie du binaire dans `~/bin` sur le cluster, puis ajout d'une ligne crontab documentée dans le README.

## Open Questions

- Faut-il aussi publier un binaire `aarch64` (pour les clusters ARM) ? Cela n'a pas d'impact sur le code, seulement sur la compilation : à décider au moment de la publication.
