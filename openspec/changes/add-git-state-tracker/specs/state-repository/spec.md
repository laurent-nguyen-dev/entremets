# Spec Delta

## Purpose

Faire d'un dépôt Git à la fois la configuration et la mémoire de l'outil : où il se trouve, comment il est organisé, comment l'outil le lit, le modifie et le partage entre plusieurs machines.

## ADDED Requirements

### Requirement: Dépôt passé explicitement
Toute commande d'`entremets` qui lit ou modifie un dépôt MUST recevoir ce dépôt explicitement avec l'option `--repo <emplacement>`, où l'emplacement est tout ce que `git clone` accepte (URL distante ou chemin d'un dépôt). L'outil MUST NOT utiliser le dossier courant comme dépôt implicite.

#### Scenario: Option absente
- **WHEN** l'utilisateur lance `entremets scan` sans `--repo`
- **THEN** la commande explique que `--repo` est obligatoire et se termine avec le code 2, même si le dossier courant est un dépôt entremets

#### Scenario: Dépôt sur GitLab
- **WHEN** l'utilisateur lance `entremets scan --repo git@gitlab.example.org:me/cluster-x.git`
- **THEN** l'outil travaille sur ce dépôt distant

### Requirement: Travail dans un clone temporaire
Chaque exécution MUST travailler dans un clone temporaire du dépôt, propre à cette exécution, et le supprimer à la fin, en cas de succès comme d'erreur. Deux exécutions simultanées MUST NOT partager de copie de travail.

#### Scenario: Deux jobs simultanés
- **WHEN** deux jobs lancent `entremets scan` en même temps sur le même dépôt
- **THEN** chacun travaille dans son propre clone temporaire, sans erreur de verrou Git

#### Scenario: Nettoyage après erreur
- **WHEN** une exécution échoue en cours de route
- **THEN** son clone temporaire est supprimé

### Requirement: Modifications commitées et poussées
Toute commande qui modifie le dépôt MUST commiter ses modifications et les pousser vers l'emplacement d'origine avant de se terminer avec succès.

#### Scenario: Dépôt local non nu
- **WHEN** l'emplacement passé est un clone de travail ordinaire dont la branche visée est active
- **THEN** l'outil signale que Git refuse ce push et qu'il faut utiliser un dépôt nu (`git init --bare`) ou un serveur, et se termine avec le code 2

### Requirement: Organisation des branches
La branche `main` MUST contenir la configuration `entremets.toml` et le dossier `plugins/`. L'état de chaque contexte MUST être stocké sur une branche orpheline `state/<contexte>`, sans historique commun avec `main`, contenant un fichier d'état par objet.

#### Scenario: Historique d'une partition
- **WHEN** l'utilisateur lance `git log state/gpu` dans un clone du dépôt
- **THEN** il ne voit que les changements d'état constatés sur le contexte `gpu`, sans les changements de configuration

#### Scenario: Comparaison de deux contextes
- **WHEN** l'utilisateur lance `git diff state/cpu state/gpu`
- **THEN** il voit les différences d'état entre les deux contextes, objet par objet

### Requirement: Création automatique d'une branche de contexte
Si la branche `state/<contexte>` n'existe pas encore, le scan de ce contexte MUST la créer en tant que branche orpheline.

#### Scenario: Premier scan sur une partition
- **WHEN** le premier scan du contexte `cpu` a lieu
- **THEN** la branche orpheline `state/cpu` est créée et poussée avec l'état initial des objets

### Requirement: Création d'un dépôt
La commande `entremets init --repo <emplacement>` MUST initialiser un dépôt vide avec, sur `main`, un `entremets.toml` d'exemple commenté, un dossier `plugins/` et un `README.md` décrivant le dépôt, puis commiter et pousser. Elle MUST refuser un dépôt qui contient déjà des commits.

#### Scenario: Dépôt vide
- **WHEN** l'utilisateur lance `entremets init` sur un dépôt nu vide
- **THEN** la branche `main` est créée avec les trois éléments, et `entremets check` sur ce dépôt réussit

#### Scenario: Dépôt déjà utilisé
- **WHEN** l'utilisateur lance `entremets init` sur un dépôt qui contient déjà des commits
- **THEN** la commande refuse, n'y modifie rien et se termine avec le code 2

### Requirement: Reprise en cas de push concurrent
Si un push est refusé parce que la branche distante a avancé, l'outil MUST récupérer la nouvelle version de la branche, réappliquer son travail par-dessus en recalculant ce qui a réellement changé, puis réessayer, jusqu'à 5 tentatives.

#### Scenario: Deux jobs constatent le même changement
- **WHEN** deux jobs de la partition `gpu` constatent le même changement d'objet, et que le second pousse après le premier
- **THEN** le second constate que la branche distante contient déjà cet état et ne crée pas de commit en double

#### Scenario: Tentatives épuisées
- **WHEN** le push est refusé 5 fois de suite
- **THEN** l'outil le signale et se termine avec le code 2

### Requirement: Dépôt injoignable
Si le dépôt ne peut être ni cloné ni poussé (réseau, droits), l'outil MUST l'indiquer clairement sur la sortie d'erreur et se terminer avec le code 2. Les changements non poussés ne sont pas conservés : ils seront constatés de nouveau par le scan suivant.

#### Scenario: Push impossible
- **WHEN** le clone a réussi mais le push échoue parce que le serveur est devenu injoignable
- **THEN** l'outil affiche l'erreur, se termine avec le code 2, et le scan suivant réussi enregistre les mêmes changements
