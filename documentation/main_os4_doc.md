## Documentation du workflow reutilisable pour OpenShift 4 "main_os4.yml"

Ce workflow CI/CD réutilisable est conçu pour automatiser le processus de build, de test, de déploiement et de promotion des images pour des applications hébergées sur OpenShift 4. Il peut être appelé depuis d'autres projets pour simplifier et standardiser le pipeline CI/CD. Le workflow inclut la construction des images, les tests end-to-end avec Cypress, et le déploiement vers les environnements DEV, TEST et PROD avec gestion des versions SemVer lors des déploiements en production.

### Déclenchement du workflow

Le workflow est déclenché via `workflow_call`, ce qui permet à d'autres workflows de l'appeler avec des paramètres personnalisés. Il accepte les entrées suivantes :

- `service` (obligatoire) : Nom du service (ex: svc0055)
- `repo_config_name` (obligatoire) : Nom du dépôt de config
- `project_part` (obligatoire) : Partie du projet à déployer (ex : frontend, backend, api)
- `run_e2e_cypress` (optionnel) : Booléen qui spécifie si les tests Cypress doivent être exécutés (par défaut : `false`).

Les secrets suivants doivent être configurés dans le dépôt appelant ce workflow :

- `QUAY_ROBOT_PASSWORD` : Mot de passe du robot Quay pour l'authentification
- `GHCR_TOKEN` : Token pour l'accès au registre GHCR (disponible dans Keybase)
- `CYPRESS_RECORD_KEY` (optionnel) : Clé Cypress pour les tests e2e

### Structure des Jobs

#### 1. **Install**
Ce job installe les dépendances et construit l'application si les tests Cypress doivent être exécutés.

- Il vérifie le code source et exécute `npm install` avec l'option `--force` pour installer les dépendances nécessaires.
- Il compile ensuite l'application et sauvegarde le répertoire de build en tant qu'artifact pour une utilisation ultérieure dans les tests Cypress.

#### 2. **E2E Testing**
Ce job exécute les tests end-to-end avec Cypress en mode parallèle.

- Le répertoire de build est téléchargé et l'application est servie localement.
- Cypress est configuré pour exécuter des tests en parallèle sur plusieurs containers (`matrix.containers`).
- Les résultats des tests sont enregistrés dans le tableau de bord Cypress.

#### 3. **Deploy to DEV**
Ce job déploie l'application dans l'environnement de développement (DEV)

- Il se connecte aux registres Quay et GHCR
- Il construit une image Docker à partir du code source, la tague avec le SHA du commit et la pousse sur Quay (quay-its.epfl.ch). Le SHA reste inchangé pour un commit donné, même si le workflow est ré-exécuté ultérieurement (par exemple, en re-déclenchant un job ou en utilisant des workflows manuels).
- Le dépôt de config associé est mis à jour avec la nouvelle version de l'image Docker. Exemple de message de commit "[auto] update DEV frontend image (866f01656562efab191f64e780b41c4f70fc6c21)"

#### 4. **Deploy to TEST**
Ce job: 
- est exécuté seulement après que le job deploy-dev soit terminé avec succès et si le commit est fusionné dans la branche principale (`main` ou `master`)
- met à jour le dépôt de config avec la nouvelle version de l'image pour l'environnement TEST (taggé avec le SHA)

Cela permet de déployer l'application dans l'environnement de TEST)

#### 5. **Tag and Release**
Ce job gère la publication des versions en prod suivant SemVer.

- Si un commit est poussé dans `main` ou `master`, une nouvelle version SemVer est générée.
- Le workflow utilise l'action `autotag` pour créer un tag basé sur le commit et déployer cette version sur GitHub en tant que release.

S'il n'y a pas de nouveaux commits depuis ce dernier tag, l'action évite de créer un nouveau tag.

Il faut utiliser les tags #major, #minor, ou #patch dans les messages de commit, et l'action d'[auto-tagging](https://github.com/phish108/autotag-action) augmentera les versions en conséquence. Par défaut, c'est #patch qui est appliqué si aucun tag n'est spécifié.

(amélioration : ajouter la condition "si une pull request est fusionnée dans `main` ou `master" avec if: github.event.pull_request.merged == true && (github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'))

#### 6. **Deploy to PROD**
Ce job déploie l'application dans l'environnement de PROD

- L'image Docker est taguée avec la version SemVer générée précédemment et poussée sur Quay
- Le dépôt de config est mis à jour

### Nota Bene

#### Promotion d'Image
Les images ne sont retaguées qu'au moment du déploiement en production, ce qui garantit que les mêmes artefacts sont testés dans les environnements de développement, de test et de production.