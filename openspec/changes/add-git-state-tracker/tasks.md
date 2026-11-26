# Tasks

## 1. Mise en place du projet

- [ ] 1.1 Créer la crate binaire `entremets` (Cargo.toml avec les dépendances du design, `src/main.rs`, `.gitignore` excluant `target/`), et vérifier que `cargo build` réussit et que `git status` n'affiche pas `target/`
- [ ] 1.2 Définir avec clap les sous-commandes `init`, `check`, `scan`, `plugins`, `objects`, `add` et `remove`, avec `--repo` obligatoire, et vérifier par un test `assert_cmd` qu'une commande sans `--repo` se termine avec le code 2
- [ ] 1.3 Mettre en place la CI GitHub Actions (`cargo fmt --check`, `cargo clippy -D warnings`, `cargo test`, compilation musl), et vérifier qu'elle passe sur la branche

## 2. Configuration

- [ ] 2.1 Implémenter l'analyse des durées (`30s`, `2m`, `1h`), et vérifier par des tests unitaires (valeurs valides et invalides)
- [ ] 2.2 Implémenter l'analyseur d'ensembles de nœuds (intervalles, zéros de tête, groupes multiples, listes) et le test d'appartenance, et vérifier par des tests unitaires couvrant les scénarios de la spec `configuration`
- [ ] 2.3 Implémenter le chargement de `entremets.toml` en types Rust avec positions (`toml_edit`), valeurs par défaut comprises, et vérifier par des tests sur des fichiers d'exemple dans `tests/fixtures/`
- [ ] 2.4 Implémenter la détermination du contexte (nom d'hôte court, `--context`, hôte inconnu, contextes qui se chevauchent), et vérifier par des tests unitaires

## 3. Plugins

- [ ] 3.1 Définir le format de plugin (params, command, exec, variants, capture) et son chargement (plugins fournis embarqués, plugins du dépôt prioritaires), et vérifier par des tests de chargement et de remplacement
- [ ] 3.2 Implémenter la substitution des paramètres avec protection shell, et vérifier par des tests incluant espaces, apostrophes et `;`
- [ ] 3.3 Implémenter l'exécuteur (sh -c ou script, arguments et variables `ENTREMETS_PARAM_*`, capture stdout/stderr/both, groupe de processus, délai avec SIGTERM puis SIGKILL, résultat ok, erreur ou timeout), et vérifier par des tests avec `sleep`, `false` et un sous-processus orphelin
- [ ] 3.4 Implémenter la sélection de variante (première condition `when` qui réussit, sinon « no applicable variant »), et vérifier par des tests unitaires
- [ ] 3.5 Écrire les plugins fournis `command`, `kernel`, `os`, `env`, `text-file`, `binary-file` et `module`, et vérifier par des tests d'intégration dans le conteneur (le cas `module` étant testé avec un faux `LMOD_CMD`)
- [ ] 3.6 Écrire le plugin exécutable `package.sh` (RPM avec époque normalisée, dpkg, pacman, `not installed`), et vérifier par des tests avec de faux `rpm`, `dpkg-query` et `pacman` placés dans le PATH
- [ ] 3.7 Compléter la validation (`check`) avec les règles des plugins (type connu, paramètres requis et inconnus, contextes, durées) et des messages `fichier:ligne:colonne: cause`, et vérifier par des tests de sortie sur des configurations invalides

## 4. Couche Git

- [ ] 4.1 Implémenter l'enveloppe autour de la commande `git` (vérification de version, clone minimal de `main`, récupération ou création orpheline de `state/<ctx>` dans un worktree, nettoyage garanti), et vérifier par des tests sur un dépôt nu temporaire
- [ ] 4.2 Implémenter la création de commits avec trailers, la détection de l'identité d'auteur et le push, avec un message explicite pour un dépôt local non nu, et vérifier par des tests d'intégration
- [ ] 4.3 Implémenter `init` (dépôt vide exigé, `entremets.toml` d'exemple commenté, `plugins/`, `README.md` du dépôt), et vérifier qu'`init` suivi de `check` réussit et qu'un second `init` est refusé avec le code 2

## 5. Scan

- [ ] 5.1 Implémenter la sélection des objets (filtre `contexts`) et le relevé parallèle limité à `jobs`, et vérifier par un test de durée avec 8 objets `sleep 1`
- [ ] 5.2 Implémenter la comparaison avec la branche d'état, l'écriture des fichiers, les commits par objet (résumés `a -> b`, `initial state`, `error (exit N)`, `changed`) et la suppression des objets retirés, et vérifier par des tests d'intégration couvrant chaque scénario de la spec `object-scan`
- [ ] 5.3 Implémenter la reprise des push concurrents (réalignement, recalcul, 5 tentatives), et vérifier par un test lançant deux scans simultanés sur le même dépôt nu, sans commit en double
- [ ] 5.4 Câbler `scan` de bout en bout (validation préalable, sortie une ligne par commit, codes 0, 1 et 2, avertissements de timeout), et vérifier par des tests `assert_cmd`

## 6. Gestion des objets

- [ ] 6.1 Implémenter `objects` et `plugins [nom]` (liste, provenance, détail des paramètres, exemple de bloc), et vérifier par des tests de sortie
- [ ] 6.2 Implémenter `add` et `remove` avec validation préalable, édition `toml_edit` préservant les commentaires et messages de commit contenant le bloc TOML, et vérifier par des tests comparant le diff du commit et le message

## 7. Notification

- [ ] 7.1 Implémenter la construction du mail (sujet, corps, objets critiques seulement) et son envoi par `sendmail -t` (ou `notification.sendmail`), uniquement après un push réussi, et vérifier par des tests avec un faux `sendmail` qui enregistre son entrée (cas critique, normal seul, push en échec, sendmail absent, aucun destinataire)

## 8. Publication et documentation

- [ ] 8.1 Ajouter les licences `LICENSE-MIT` et `LICENSE-CeCILL-B`, `license = "MIT OR CECILL-B"` dans Cargo.toml et des en-têtes SPDX, et vérifier que `cargo package --list` inclut les deux licences
- [ ] 8.2 Rédiger le README en anglais (principe, installation, démarrage rapide, exemples crontab et sbatch, référence de la configuration et des plugins, écriture d'un plugin, sécurité, limites), et vérifier en exécutant les commandes du démarrage rapide sur un dépôt nu
- [ ] 8.3 Ajouter `CONTRIBUTING.md`, et vérifier qu'il décrit le flux OpenSpec, les commandes de test et de lint
- [ ] 8.4 Vérifier que le binaire musl est statique (`file` et `ldd`) et faire un essai de bout en bout : `init`, `add`, deux scans dans deux contextes, `remove`, puis un scan, en contrôlant l'historique avec `git log` et les codes de retour

## Workflow follow-up

- Tester le binaire sur un vrai cluster (nœud de connexion, puis jobs sur deux partitions).
- Traduire en anglais tous les documents OpenSpec (`openspec/`) avant la publication, et passer `openspec/config.yaml` en anglais.
- Archiver le changement (`/openspec-archive-change`) une fois validé.
