# DEVOPS - TP 02 - CI/CD
## First steps into the CI world

Tout d'abord, pour tester l'application, on se place au niveau du fichier pom.xml et on lance :
```
mvn clean verify
```

On créé le repo git à partir du TP 1 comme ceci 
```
git init
git add .
git commit -m "First commit"
git branch -M main
git remote add origin https://github.com/Toufub/DEVOPS2022-TP1.git
git push -u origin main

```

Par la suite, pour gérer les workflows de Github on a ajouté les dossiers nécessaires au projet et le fichier main.yml :
```
mkdir .github
mkdir .github/workflows
touch main.yml
```

Voici le contenu de ce fichier main.yml :
```
name:  CI  devops  2022  CPE
on:
  # on liste les branches sur lesquelles faire les tests
  push:
    branches:
      - main
      - develop
  pull_request:
jobs:
  test-backend:
    runs-on:  ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3  
      - uses: actions/checkout@v2.3.3
      #on se sert d'une action pour setup un JDK11  
      - name:  Set  up  JDK  11
        uses: actions/setup-java@v2
        with:
          distribution: 'zulu'
          java-version: '11'
      #on run et on vérifie avec maven le projet en spécifiant l'emplacement du pom.xml
      - name:  Build  and  test  with  Maven
        run:  mvn -B verify --file backend-API/simple-api/pom.xml

```

## First steps into the CD world

Pour ajouter des *secrets* à Github actions, on se rend sur Github dans : Settings/Secrets/Actions.
Après les avoir ajouté, on va ajouter la construction et la publication des images Docker dans les actions :
```
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest
    # steps to perform in job
    steps:
      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{secrets.DOCKERHUB_TOKEN}}
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Build image and push backend
        uses: docker/build-push-action@v2
        with:
        # relative path to the place where source code with Dockerfile is located
          context: ./backend-API/simple-api
        # Note: tags has to be all lower-case
          tags: ${{secrets.DOCKERHUB_USERNAME}}/backend-api
          push: ${{ github.ref == 'refs/heads/main' }}
      - name: Build image and push database
        uses: docker/build-push-action@v2
        with:
          context: ./postgres
          tags: ${{secrets.DOCKERHUB_USERNAME}}/postgres
          push: ${{ github.ref == 'refs/heads/main' }}
      - name: Build image and push httpd
        uses: docker/build-push-action@v2
        with:
          context: ./http_basic
          tags: ${{secrets.DOCKERHUB_USERNAME}}/http_basic
          push: ${{ github.ref == 'refs/heads/main' }}
```

Pour que cela fonctionne il ne faut pas oublier de se connecter à Docker, d'où l'étape "login to dockerhub"

## Setup Quality Gate

Pour relier la quality gate de sonarcloud à github actions j'ai modifié l'étape test-backend comme ceci :
```
  test-backend:
    runs-on:  ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3  
      - uses: actions/checkout@v2.3.3
      #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11  
      - name:  Set  up  JDK  11
        uses: actions/setup-java@v2
        with:
          distribution: 'zulu'
          java-version: '11'
      #finally build your app with the latest command  
      - name:  Build  and  test  with  Maven
        run:  mvn -B verify sonar:sonar -Dsonar.projectKey=Toufub_DEVOPS2022-TP1 -Dsonar.organization=toufub -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{secrets.SONAR_TOKEN}} --file backend-api-2/simple-api-main/simple-api/pom.xml
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # necessaire pour SONAR_CLOUD
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

Attention, il faut bien penser à ajouter le Github token aux variables d'environnement pour SONAR CLOUD sinon ca ne fonctionne pas, pas besoin de le renseigner auprès de github actions puisqu'il est directement accessible.

## Split pipeline

Pour éviter de refaire tous les test à chaque push on peut fixer le working directory de test-backend :
```
  env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          working-directory: ./backend-api-2/simple-api-main/simple-api
```
