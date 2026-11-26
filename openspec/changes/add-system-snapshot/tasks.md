# Tasks

## 1. Mise en place du projet

- [ ] 1.1 Créer la crate binaire `entremets` (Cargo.toml, `src/main.rs`) avec les dépendances clap, serde, serde_json et anyhow, et vérifier que `cargo build` réussit
- [ ] 1.2 Définir les sous-commandes `scan`, `list` et `diff` avec leurs options (`--store`, `--host`, `--quiet`, `--json`) et vérifier que `entremets --help` les affiche
- [ ] 1.3 Ajouter un `.gitignore` (target/) et vérifier que `git status` n'affiche pas `target/` après une compilation

## 2. Modèle d'instantané et stockage

- [ ] 2.1 Définir le type `Snapshot` (format_version, host, id, horodatage, sources ok/unavailable) avec sa sérialisation JSON, et vérifier par un test aller-retour JSON
- [ ] 2.2 Implémenter la génération de l'identifiant UTC `AAAAMMJJTHHMMSSZ` depuis `SystemTime`, et vérifier par des tests unitaires sur des dates connues (y compris les années bissextiles)
- [ ] 2.3 Implémenter la résolution du répertoire de stockage (`--store` > `ENTREMETS_DIR` > `XDG_DATA_HOME` > `~/.local/share`), et vérifier par des tests couvrant chaque cas
- [ ] 2.4 Implémenter l'écriture atomique dans `<store>/<host>/<id>.json` et la lecture avec refus d'un `format_version` inconnu (code 2), et vérifier par des tests avec `tempfile`
- [ ] 2.5 Implémenter le listage trié des instantanés d'un hôte et la résolution d'un identifiant ou d'un préfixe (unique, ambigu, inconnu), et vérifier par des tests unitaires

## 3. Collecteurs

- [ ] 3.1 Définir le trait `Collector` et le mécanisme de commande externe avec délai maximal, et vérifier par un test qu'une commande trop longue renvoie « indisponible »
- [ ] 3.2 Implémenter le collecteur des informations de base (noyau, os-release, glibc, nom d'hôte court), et vérifier par des tests d'analyse sur des fichiers os-release d'exemple et par un relevé réel dans le conteneur
- [ ] 3.3 Implémenter le collecteur du pilote NVIDIA (`/proc/driver/nvidia/version`, avec `nvidia-smi` en repli), et vérifier par un test d'analyse sur une fixture et par le cas « absent » sur la machine de développement
- [ ] 3.4 Implémenter le collecteur de paquets RPM et dpkg (gestion de l'époque `(none)`), et vérifier par des tests d'analyse sur des fixtures `tests/fixtures/rpm.txt` et `dpkg.txt` et par le cas « aucun gestionnaire disponible »

## 4. Commandes scan et list

- [ ] 4.1 Câbler `entremets scan` (collecte, enregistrement, affichage de l'identifiant), et vérifier par un test d'intégration qu'un fichier JSON valide est créé dans un répertoire temporaire
- [ ] 4.2 Câbler `entremets list` (avec `--host`), y compris le nombre de différences par rapport à l'instantané précédent et le cas « aucun instantané », et vérifier par un test d'intégration sur des instantanés d'exemple

## 5. Comparaison et rapport

- [ ] 5.1 Implémenter `DiffReport` (informations de base modifiées, paquets ajoutés, supprimés ou changés, sources devenues indisponibles) et vérifier par des tests unitaires couvrant chaque scénario de la spec `snapshot-diff`
- [ ] 5.2 Implémenter le rendu texte groupé par catégorie et le rendu `--json`, et vérifier par des tests de sortie (JSON valide, contenu identique)
- [ ] 5.3 Câbler `entremets diff [A] [B]` (0, 1 ou 2 arguments) avec les codes de sortie 0, 1 et 2, et vérifier par des tests d'intégration
- [ ] 5.4 Ajouter à `scan` la comparaison avec l'instantané précédent, le code de sortie et le mode `--quiet`, et vérifier par des tests d'intégration (premier relevé, sans changement, avec changement)

## 6. Distribution et documentation

- [ ] 6.1 Vérifier la compilation statique `x86_64-unknown-linux-musl` et confirmer avec `file` ou `ldd` que le binaire est statique
- [ ] 6.2 Rédiger le README en français (installation, utilisation, exemple de crontab, limites de la v1), et vérifier en exécutant les commandes documentées
- [ ] 6.3 Faire un essai de bout en bout : deux scans, modification simulée d'un instantané, puis `diff` et `list`, et vérifier les sorties et les codes de retour

## Workflow follow-up

- Relire le code et tester le binaire sur un vrai nœud du cluster.
- Archiver le changement (`/openspec-archive-change`) une fois validé, pour fusionner les specs dans `openspec/specs/`.
