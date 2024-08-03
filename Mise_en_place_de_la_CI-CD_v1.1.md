# Mise en place de la CI/CD

URL du dépôt GitHub : https://github.com/davipro34/BobApp-CI-CD

URL DockerHub front : https://hub.docker.com/repository/docker/davipro/bobapp-front/general
URL DockerHub back : https://hub.docker.com/repository/docker/davipro/bobapp-back/general

URL SonarCloud : https://sonarcloud.io/projects

## 1. Les étapes des GitHub Actions

### Backend CI/CD Pipeline

#### Fichier: `.github/workflows/back_cicd.yml`

**Nom du Workflow**: BackEnd CI CD

**Déclencheurs**:
- `push` sur les chemins `back/**` et `.github/workflows/**` dans la branche `main`.
- `pull_request` sur les chemins `back/**` et `.github/workflows/**` dans la branche `main`.

**Jobs**:

1. **backend_test_and_coverage**
   - **runs-on**: `ubuntu-latest`
   - **Objectif**: Tester le backend, vérifier la couverture de code et analyser la qualité du code avec SonarCloud.

   **Étapes**:
   - **Checkout**: Récupère le code source du repository.
     ```yaml
     - name: Checkout
       uses: actions/checkout@v4
     ```
   - **Set Up JDK 17**: Installe JDK 17 pour compiler et exécuter les tests Java.
     ```yaml
     - name: Set Up JDK 17
       uses: actions/setup-java@v4
       with:
         java-version: '17'
         distribution: 'adopt'
     ```
   - **Cache Maven Packages**: Cache les dépendances Maven pour accélérer les builds futurs.
     ```yaml
     - name: Cache Maven Packages
       uses: actions/cache@v4
       with:
         path: ~/.m2
         key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
         restore-keys: |
           ${{ runner.os }}-m2
     ```
   - **Cache Build Output**: Cache les résultats de la compilation Maven.
     ```yaml
     - name: Cache Build Output
       uses: actions/cache@v4
       with:
         path: target
         key: ${{ runner.os }}-build-${{ hashFiles('**/pom.xml') }}
         restore-keys: |
           ${{ runner.os }}-build-
     ```
   - **Build and Test with Maven**: Compile le projet et exécute les tests.
     ```yaml
     - name: Build and Test with Maven
       run: mvn -B clean verify
     ```
   - **Upload Jacoco Report**: Télécharge le rapport de couverture de code Jacoco.
     ```yaml
     - name: Upload Jacoco Report
       uses: actions/upload-artifact@v4
       with:
         name: jacoco-report
         path: back/target/site/jacoco/
         overwrite: true
         if-no-files-found: error
     ```
   - **Check sonar-project.properties**: Vérifie le fichier de configuration SonarQube.
     ```yaml
     - name: Check sonar-project.properties
       run: cat sonar-project.properties
     ```
   - **SonarCloud Scan**: Analyse le code avec SonarCloud.
     ```yaml
     - name: SonarCloud Scan
       env:
         SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
       run: mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:4.0.0.4121:sonar
     ```

2. **docker_build_and_push**
   - **needs**: `backend_test_and_coverage`
   - **runs-on**: `ubuntu-latest`
   - **Objectif**: Construire et déployer l'image Docker du backend.

   **Étapes**:
   - **Checkout**: Récupère le code source du repository.
     ```yaml
     - name: Checkout
       uses: actions/checkout@v4
     ```
   - **Set Up Docker Buildx**: Configure Docker Buildx.
     ```yaml
     - name: Set Up Docker Buildx
       uses: docker/setup-buildx-action@v3.3.0
     ```
   - **Login to Docker Hub**: Connecte à Docker Hub.
     ```yaml
     - name: Login to Docker Hub
       uses: docker/login-action@v3.2.0
       with:
         username: ${{ secrets.DOCKER_USERNAME }}
         password: ${{ secrets.DOCKER_PASSWORD }}
     ```
   - **Build and Push Backend Docker Image**: Construit et pousse l'image Docker du backend.
     ```yaml
     - name: Build and Push Backend Docker Image
       uses: docker/build-push-action@v6.2.0
       with:
         context: ./back
         file: ./back/Dockerfile
         push: true
         tags: ${{ secrets.DOCKER_USERNAME }}/bobapp-back:latest
     ```

### Frontend CI/CD Pipeline

#### Fichier: `.github/workflows/front_cicd.yml`

**Nom du Workflow**: FrontEnd CI CD

**Déclencheurs**:
- `push` sur les chemins `front/**` et `.github/workflows/**` dans la branche `main`.
- `pull_request` sur les chemins `front/**` et `.github/workflows/**` dans la branche `main`.

**Jobs**:

1. **frontend_build_test_analyze**
   - **runs-on**: `ubuntu-latest`
   - **Objectif**: Construire le frontend, exécuter les tests et analyser la qualité du code avec SonarCloud.

   **Étapes**:
   - **Checkout**: Récupère le code source du repository.
     ```yaml
     - name: Checkout
       uses: actions/checkout@v4
     ```
   - **Use Node.js ${{ matrix.node-version }}**: Installe Node.js pour le build et les tests.
     ```yaml
     - name: Use Node.js ${{ matrix.node-version }}
       uses: actions/setup-node@v4
       with:
         node-version: ${{ matrix.node-version }}
     ```
   - **Cache Node Modules**: Cache les modules Node.js pour accélérer les builds futurs.
     ```yaml
     - name: Cache Node Modules
       uses: actions/cache@v4
       with:
         path: ~/.npm
         key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
         restore-keys: |
           ${{ runner.os }}-node-
     ```
   - **Install Dependencies**: Installe les dépendances nécessaires au projet.
     ```yaml
     - name: Install Dependencies
       run: npm ci
     ```
   - **Build Angular Project**: Compile le projet Angular.
     ```yaml
     - name: Build Angular Project
       run: npm run build
     ```
   - **Run Tests and Generate Code Coverage Report**: Exécute les tests et génère le rapport de couverture de code.
     ```yaml
     - name: Run Tests and Generate Code Coverage Report
       run: npm run test -- --no-watch --no-progress --browsers=ChromeHeadless --code-coverage
     ```
   - **Upload Code Coverage Report**: Télécharge le rapport de couverture de code.
     ```yaml
     - name: Upload Code Coverage Report
       uses: actions/upload-artifact@v4
       with:
         name: front-coverage-report
         path: front/coverage/
         overwrite: true
         if-no-files-found: error
     ```
   - **SonarCloud Scan**: Analyse le code avec SonarCloud.
     ```yaml
     - name: SonarCloud Scan
       uses: SonarSource/sonarcloud-github-action@master
       with:
         projectBaseDir: front
       env:
         SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
         GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
     ```

2. **docker_build_and_push**
   - **needs**: `frontend_build_test_analyze`
   - **runs-on**: `ubuntu-latest`
   - **Objectif**: Construire et déployer l'image Docker du frontend.

   **Étapes**:
   - **Checkout**: Récupère le code source du repository.
     ```yaml
     - name: Checkout
       uses: actions/checkout@v4
     ```
   - **Set Up Docker Buildx**: Configure Docker Buildx.
     ```yaml
     - name: Set Up Docker Buildx
       uses: docker/setup-buildx-action@v3.3.0
     ```
   - **Login to Docker Hub**: Connecte à Docker Hub.
     ```yaml
     - name: Login to Docker Hub
       uses: docker/login-action@v3.2.0
       with:
         username: ${{ secrets.DOCKER_USERNAME }}
         password: ${{ secrets.DOCKER_PASSWORD }}
     ```
   - **Build and Push Frontend Docker Image**: Construit et pousse l'image Docker du frontend.
     ```yaml
     - name: Build and Push Frontend Docker Image
       uses: docker/build-push-action@v6.2.0
       with:
         context: ./front
         file: ./front/Dockerfile
         push: true
         tags: ${{ secrets.DOCKER_USERNAME }}/bobapp-front:latest
     ```


## 2. Key Performance Indicators (KPIs)

Pour améliorer la qualité de BobApp et assurer une gestion efficace du code, je propose les KPIs suivants basés sur le Quality Gate "Sonar Way" de SonarQube. Ces KPIs permettront de surveiller et de maintenir la qualité du code et de réduire les bugs et vulnérabilités.

### 1. Couverture de Code (Code Coverage)

**Description**: Ce KPI mesure le pourcentage de code source couvert par les tests. Une couverture de code élevée indique que la majorité du code a été testée, ce qui réduit les risques de bugs non détectés.

**Seuil Minimum**: 80%

**Justification**: Le Quality Gate "Sonar Way" recommande une couverture minimale de 80%. Ce seuil est généralement considéré comme un bon équilibre entre l'effort de test et la détection de bugs potentiels. Une couverture de 80% garantit que la majorité du code est testée tout en laissant une certaine flexibilité pour les parties de code difficilement testables.

### 2. Évaluation de la Fiabilité (Reliability Rating)

**Description**: Ce KPI évalue la fiabilité du code en fonction des bugs détectés. SonarQube attribue une note de A à E, A étant la meilleure.

**Seuil Minimum**: A

**Justification**: Une évaluation de la fiabilité de A indique qu'aucun nouveau bug critique n'a été introduit dans le code. Cela est essentiel pour maintenir la stabilité de l'application et offrir une expérience utilisateur positive. Les utilisateurs se plaignent actuellement de bugs récurrents, et ce KPI aidera à réduire ces problèmes en assurant que le nouveau code soit exempt de bugs majeurs.

### 3. Évaluation de la Sécurité (Security Rating)

**Description**: Ce KPI évalue la sécurité du code en fonction des vulnérabilités détectées. SonarQube attribue une note de A à E, A étant la meilleure.

**Seuil Minimum**: A

**Justification**: Une évaluation de la sécurité de A signifie qu'aucune nouvelle vulnérabilité critique n'a été introduite. La sécurité est cruciale pour protéger les données des utilisateurs et maintenir la confiance dans l'application. En s'assurant que le nouveau code est sécurisé, nous pouvons prévenir les failles de sécurité qui pourraient entraîner des pertes de données ou des violations de la vie privée.

### 4. Évaluation de la Maintenabilité (Maintainability Rating)

**Description**: Ce KPI mesure la maintenabilité du code en fonction de la dette technique détectée. SonarQube attribue une note de A à E, A étant la meilleure.

**Seuil Minimum**: A

**Justification**: Une évaluation de la maintenabilité de A indique que le nouveau code est bien structuré et facile à maintenir. Cela est essentiel pour faciliter les futures modifications et améliorations du code. Une bonne maintenabilité réduit également le temps nécessaire pour corriger les bugs et implémenter de nouvelles fonctionnalités, ce qui est crucial pour Bob qui a peu de temps disponible pour gérer l'application.

### 5. Révision des Points Sensibles de Sécurité (Security Hotspots)

**Description**: Ce KPI s'assure que tous les points sensibles de sécurité (Security Hotspots) identifiés dans le nouveau code ont été examinés.

**Seuil Minimum**: 100%

**Justification**: Examiner tous les points sensibles de sécurité permet de s'assurer que des décisions conscientes ont été prises pour atténuer les risques potentiels. Cela améliore la sécurité globale du code et aide à identifier et à résoudre les problèmes avant qu'ils ne deviennent des vulnérabilités exploitables.

### Conclusion

En mettant en place ces KPIs, nous assurons que BobApp maintient un haut niveau de qualité, de sécurité et de maintenabilité. Ces indicateurs permettent de détecter et de corriger les problèmes tôt dans le cycle de développement, réduisant ainsi les bugs en production et améliorant l'expérience utilisateur.

## 3. L’analyse des métriques et des retours utilisateurs

### Analyse des Métriques

L'analyse des métriques issues de SonarQube et des rapports de couverture de code fournit des informations précieuses sur la qualité et la fiabilité du code. Voici un résumé des principales métriques et leur interprétation :

#### Backend

**Rapport de couverture de code Jacoco**:
- **Couverture des Instructions**: 32%
- **Couverture des Branches**: 50%
- **Complexité Cyclomatique**: 15
- **Lignes Manquantes**: 45
- **Méthodes Manquantes**: 18
- **Classes Manquantes**: 6

**Analyse**: La couverture des instructions pour le backend est de 32%, ce qui est en dessous du seuil recommandé de 80%. La couverture des branches est également faible à 50%. Ces métriques indiquent qu'une grande partie du code backend n'est pas testée, augmentant le risque de bugs en production. L'accent doit être mis sur l'augmentation de la couverture de test, en particulier pour les classes et méthodes critiques.

#### Frontend

**Rapport de couverture de code Istanbul**:
- **Couverture des Déclarations**: 76.92%
- **Couverture des Branches**: 100%
- **Couverture des Fonctions**: 57.14%
- **Couverture des Lignes**: 83.33%

**Analyse**: La couverture des déclarations pour le frontend est de 76.92%, ce qui est proche du seuil recommandé de 80%. Cependant, la couverture des fonctions est relativement faible à 57.14%. Il est important d'augmenter la couverture des tests pour les fonctions afin de s'assurer que toutes les fonctionnalités critiques du frontend sont testées.

### Analyse des Retours Utilisateurs

Les retours des utilisateurs fournissent des informations qualitatives importantes qui complètent les données quantitatives des métriques. Voici une synthèse des principaux retours utilisateurs et des problèmes identifiés :

#### Commentaires des utilisateurs

1. **Bug inexistant de suggestion de blague**
   - **Note** : Une étoile
   - **Problème** : Les utilisateurs mentionnent un bug avec un bouton de suggestion de blague, mais cette fonctionnalité n'existe pas dans l'application actuelle.
   - **Action** : Bien que cette fonctionnalité n'existe pas, il pourrait être intéressant d'ajouter une fonctionnalité de suggestion de blague à l'avenir.

2. **Bug inexistant de post de vidéo**
   - **Note** : Deux étoiles
   - **Problème** : Les utilisateurs signalent un bug persistant sur le post de vidéo, mais cette fonctionnalité n'existe pas non plus.
   - **Action** : Cette fonctionnalité pourrait également être envisagée pour de futures mises à jour.

3. **Absence de réponses par email**
   - **Note** : Une étoile
   - **Problème** : Les utilisateurs ne reçoivent plus de réponses à leurs emails.
   - **Action Prioritaire** : Pour maintenir l'engagement des utilisateurs les encourager à créer des issues sur GitHub pour un suivi plus efficace des problèmes et des demandes.

4. **Déception et suppression de l'application des favoris**
   - **Note** : Deux étoiles
   - **Problème** : Les utilisateurs sont déçus par les bugs récurrents et ont supprimé l'application de leurs favoris.
   - **Action Prioritaire** : Améliorer la qualité générale de l'application en réduisant les bugs et en introduisant de nouvelles fonctionnalités pour regagner la confiance des utilisateurs.

### Actions Prioritaires

En se basant sur les métriques de qualité et les retours utilisateurs, voici les actions prioritaires à mener pour améliorer BobApp :

1. **Résolution des Bugs Existants** :
   - Corriger les bugs actuels qui affectent l'expérience utilisateur, même si certains bugs rapportés par les utilisateurs concernent des fonctionnalités non existantes, il est crucial de s'assurer que toutes les fonctionnalités actuelles fonctionnent correctement.

2. **Amélioration des réponses au Support** :
   - Améliorer la réactivité du support en traitant les demandes sous formes d'"issues" directement dans GitHub.

3. **Augmentation de la Couverture de Code** :
   - Pour le backend, augmenter la couverture des instructions et des branches en ajoutant des tests unitaires et d'intégration supplémentaires.
   - Pour le frontend, améliorer la couverture des fonctions en ajoutant des tests pour les fonctionnalités critiques.

4. **Engagement des Utilisateurs** :
   - Introduire de nouvelles fonctionnalités, telles que la suggestion de blague et le post de vidéo, pour attirer et retenir les utilisateurs.
   - Assurer une communication régulière et transparente avec les utilisateurs pour maintenir leur engagement et leur satisfaction.


5. **Tests en production** :
   - Mettre en place une procédure de test en production afin de veiller à ce que la version déployée en production soit fonctionnelle. La version actuelle déployée par Docker intégrant le serveur NginX présente un bug critique qui empêche l'affichage du contenu de la blague.


En mettant en œuvre ces actions, nous pouvons améliorer la qualité et la fiabilité de BobApp, tout en répondant aux attentes des utilisateurs et en renforçant leur confiance dans l'application.
