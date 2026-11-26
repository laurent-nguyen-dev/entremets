# Design

## Context

Projet nouveau : seule l'infrastructure OpenSpec existe dans le dépôt. La motivation est dans proposal.md (section Why) et le comportement attendu dans les specs. Contraintes d'environnement :
- l'outil tourne sur des nœuds de connexion et de calcul de clusters HPC, sans droits root ;
- la glibc peut être ancienne, et l'outillage disponible minimal (`git`, `sh`, coreutils) ;
- plusieurs jobs peuvent scanner en même temps le même dépôt ;
- les nœuds de calcul n'ont pas toujours accès au réseau extérieur, ni à `sendmail` ;
- l'outil est destiné à être publié en open source. Le code, les messages, la configuration et la documentation de l'outil sont donc en anglais.

## Goals / Non-Goals

**Goals:**
- Un binaire unique, copiable tel quel sur n'importe quel nœud Linux x86_64.
- Des états stockés en texte brut, pour que `git diff`, `git log` et les interfaces GitLab montrent directement ce qui a changé.
- Ajouter un type d'objet sans recompiler l'outil.

**Non-Goals:**
- Aucune base de données ni aucun cache persistant sur la machine scannée : le dépôt est la seule mémoire.
- Pas de bibliothèque Git embarquée (voir Décisions).
- Pas de support de la syntaxe de groupes ClusterShell (`@groupe`) ni de l'appel à ClusterShell lui-même : seule la syntaxe des ensembles de nœuds est reprise.

## Decisions

### Binaire statique musl
Compilation pour la cible `x86_64-unknown-linux-musl`, qui donne un binaire indépendant de la glibc du cluster.
- *Alternative* : cible gnu. Un binaire compilé sur une machine récente refuse alors de démarrer sur une glibc ancienne.

### Git via la commande `git`, pas via libgit2
L'outil appelle la commande `git` installée sur la machine.
- *Pourquoi* : elle réutilise telle quelle la configuration de l'utilisateur (clés SSH, `~/.ssh/config`, helpers d'identifiants, proxy), elle est présente sur tous les clusters, et elle évite de lier OpenSSL et libssh2 en statique.
- *Alternative* : la crate `git2` (libgit2). Elle offre une API plus propre, mais rend la compilation statique et l'authentification SSH plus fragiles.
- *Version minimale visée* : git 2.x (RHEL 7 fournit 1.8 ; `git` 2.x est en général disponible en module). À vérifier au démarrage, avec un message clair.

### Clone temporaire minimal
Pour un scan : `git clone --depth 1 --single-branch --branch main` dans un répertoire temporaire (`$TMPDIR`, à défaut `/tmp`), puis `git fetch --depth 1 origin state/<ctx>` et un `git worktree` pour la branche d'état. Si la branche n'existe pas, elle est créée avec `git checkout --orphan`. Le répertoire est supprimé par un garde (`Drop`) en fin d'exécution, y compris en cas d'erreur. Les commandes `add`, `remove` et `init` ne manipulent que `main`.

### Organisation du dépôt
```
main                         state/<ctx>   (orpheline)
├── entremets.toml           ├── kernel
├── plugins/                 ├── openmpi
│   ├── package.toml         └── slurm-conf
│   └── package.sh
└── README.md
```
Un fichier d'état par objet, nommé exactement comme l'objet, sans extension, à la racine de la branche d'état. Le contenu est l'état brut. Une erreur s'écrit :
```
ERROR: exit 1
<sortie d'erreur>
```

### Commits et identité
- Les commits d'état sont créés séquentiellement après la fin de tous les relevés parallèles, car Git ne permet pas deux commits simultanés dans un même dépôt.
- Les lignes `Node:`, `Context:` et `Plugin:` sont des « trailers » Git standard (`git interpret-trailers`, `git log --format=%(trailers)`).
- Auteur : la configuration Git de l'utilisateur (`user.name` et `user.email`) si elle existe, sinon `entremets <entremets@<hostname>>`.

### Reprise en cas de push concurrent
Les états relevés sont gardés en mémoire. En cas de refus du push, l'outil récupère la branche distante, s'y réaligne (`reset --hard`), recompare chaque état en mémoire à la nouvelle version, recrée les commits encore nécessaires et repousse. L'opération est tentée au plus 5 fois, avec une attente aléatoire croissante. On évite ainsi le rebase, qui produirait des conflits ou des commits vides lorsque deux jobs constatent le même changement.

### Pas de rétention locale des changements non poussés
Le clone étant temporaire, des commits non poussés seraient perdus. C'est acceptable : l'état est relevé de nouveau à chaque scan, donc le scan suivant réussi enregistre le même changement. On perd seulement la précision sur le premier nœud et la première date de constat. Cela évite tout état persistant sur la machine scannée.

### Configuration TOML et édition préservée
- Lecture et validation : crate `toml_edit`, qui fournit les positions (ligne et colonne) nécessaires aux messages de `check`, et désérialisation `serde` dans des types Rust.
- `add` et `remove` modifient le document `toml_edit`, ce qui préserve les commentaires et la mise en forme du reste du fichier.
- Durées : entier suivi de `s`, `m` ou `h` (`30s`, `2m`), analysées par une petite fonction maison.

### Ensembles de nœuds
Un analyseur maison, testé unitairement, couvre `prefix[a-b,c]suffix`, les zéros de tête (`[001-064]` impose la largeur), plusieurs groupes de crochets (`r[1-2]n[01-10]`) et les listes séparées par des virgules au premier niveau. L'appartenance d'un nom d'hôte est testée sans développer les ensembles. Le chevauchement entre contextes est contrôlé par `check` en développant les ensembles, qui comptent au plus quelques milliers de nœuds.

### Modèle de plugin
- Les plugins fournis sont embarqués dans le binaire (`include_str!`) au même format TOML que ceux du dépôt, scripts compris. Les scripts embarqués sont écrits dans le clone temporaire avant exécution. Cela garantit que les plugins fournis sont des exemples réels du format.
- Format :
  ```toml
  description = "Installed version of a system package"
  capture = "stdout"                    # stdout | stderr | both

  [params.name]
  required = true
  description = "Package name"

  # Une des trois formes : command = "...", exec = "...", ou des variantes :
  [[variants]]
  when = "command -v rpm >/dev/null"
  command = "rpm -q --qf '%{VERSION}-%{RELEASE}\\n' {name} || echo 'not installed'"
  ```
- Substitution : `{param}` est remplacé par la valeur protégée pour le shell (entre apostrophes, avec échappement des apostrophes). Le plugin `command` est le seul cas particulier : son paramètre `command` est la commande elle-même, exécutée telle quelle.
- `package` est fourni sous forme exécutable (`package.sh`), à cause des cas particuliers : époque RPM `(none)`, paquets multi-architectures, format de pacman.
- `module` utilise des variantes : `$LMOD_CMD sh show <nom>` si `LMOD_CMD` est défini, sinon `$MODULESHOME/bin/modulecmd sh show <nom>` (ou `modulecmd`), avec `capture = "stderr"`. On évite ainsi de dépendre de la fonction shell `module`, absente dans un shell non interactif.
- `os` : `. /etc/os-release && echo "$PRETTY_NAME"`. `kernel` : `uname -r`. `env` : `printenv <nom> || echo unset`. `binary-file` : `sha256sum` et `stat -c %s`. `text-file` : `cat`.

### Exécution parallèle et délais
Un pool de threads de taille `jobs` (fils `std::thread` et canal, sans runtime asynchrone). Chaque commande est lancée dans son propre groupe de processus. À l'expiration du délai, tout le groupe reçoit `SIGTERM`, puis `SIGKILL` 2 secondes plus tard, ce qui arrête aussi les sous-processus d'un script.

### Mail
Le message RFC 5322 (en-têtes `From`, `To`, `Subject`, corps en texte UTF-8) est transmis sur l'entrée standard de `sendmail -t`. Il n'y a ni client SMTP intégré ni dépendance réseau supplémentaire.

### Dépendances Rust
`clap` (derive), `serde`, `toml_edit`, `anyhow`, `thiserror`, `libc` (groupes de processus et signaux) et `tempfile`. Pas de crate de hachage : `binary-file` s'appuie sur `sha256sum`. Pour les tests : `assert_cmd` et `predicates`, avec des dépôts nus créés dans des répertoires temporaires.

## Risks / Trade-offs

- [Commande à sortie non déterministe (horodatage, température GPU, PID dans un message d'erreur)] → commits à chaque scan. La documentation recommande d'écrire des commandes stables, et `check` ne peut pas le détecter. On pourra ajouter plus tard des filtres de sortie.
- [Nœud non homogène dans une partition] → l'historique oscille entre deux états. C'est accepté en v1 : l'oscillation, avec la ligne `Node:` de chaque commit, révèle le nœud fautif.
- [Le plugin `env` relève l'environnement du scan, qui diffère entre cron, job et session interactive] → à expliquer dans la documentation de ce plugin.
- [Quiconque peut pousser sur `main` peut exécuter des commandes sous le compte de l'utilisateur] → section sécurité dans le README : restreindre les droits du dépôt.
- [Un scan dont tous les relevés expirent ne produit rien et sort avec le code 0] → les avertissements restent sur la sortie d'erreur, que cron transmet par mail.
- [Git trop ancien sur certains clusters] → vérification de version au démarrage, avec un message clair.

## Migration Plan

Aucune migration nécessaire (première version). Déploiement : compilation musl, copie du binaire dans `~/bin`, `entremets init` sur un dépôt vide, puis ajout d'une ligne crontab et d'une ligne en tête des scripts sbatch, comme documenté dans le README.

## Open Questions

- Faut-il publier aussi un binaire `aarch64` pour les clusters ARM ? Cela ne concerne que la CI de publication.
