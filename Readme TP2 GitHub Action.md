# Compte Rendu TP2 GitHub Action

Lancement des tests (compilation + exécution) de notre API.

```
mvn clean verify
```

*What are TestContainers ?*

Testcontainers est une bibliothèque Java qui fournit des instances légères et temporaires de bases de données.

## Workflows

Un workflow Git permet de mettre à jour nos images à chaque push. On vérifie dans un premier temps les identifiants de l'utilisateur.

Pipeline sonar.yml : 

```
name: CI devops sonar 2024  
on:  
  #to begin you want to launch this job in main and develop  
  push:  
    branches:   
      - main   
  
jobs:  
  test-backend:   
    runs-on: ubuntu-22.04  
    steps:  
     #checkout your github code using actions/checkout@v2.5.0  
      - uses: actions/checkout@v2.5.0  
  
     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17  
      - name: Set up JDK 17  
        uses: actions/setup-java@v3  
        with:  
          # on utilise le même jdk que le Dockerfile  
          distribution: 'corretto'  
          # on précise la même version que le Dockerfile  
          java-version: '17'  
  
     #finally build your app with the latest command  
      - name: Build and test with Maven  
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=blue-fire7_DevOpsTPs -Dsonar.organization=blue-fire7 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file ./TP1/BackendAPI/simpleapi/simpleapi/pom.xml
```

Et lorsque la pipeline passe, on lance les trois images : Database, Backend et Reverse Proxy.

Pipeline docker.yml : 

```
name: CI devops docker 2024  
on:  
  #to begin you want to launch this job in main and develop  
  workflow_run:  
    workflows:  
      - "CI devops sonar 2024"  
    types:  
      - completed  
    branches:  
      - main  
  
jobs:  
  docker-images:  
    if: ${{github.event.workflow_run.conclusion == 'success'}}  
    runs-on: ubuntu-22.04  
    steps:  
      #checkout your github code using actions/checkout@v2.5.0  
      - name: Checkout  
        uses: actions/checkout@v2.5.0  
  
      - name: Login to DockerHub  
        uses: docker/login-action@v1  
        with:  
          username: ${{secrets.DOCKERHUB_USERNAME}}  
          password: ${{secrets.DOCKERHUB_TOKEN}}  
  
      - name: Build image and push backend  
        uses: docker/build-push-action@v3  
        with:  
          # relative path to the place where source code with Dockerfile is located  
          context: ./TP1/BackendAPI/simpleapi/simpleapi  
          # Note: tags has to be all lower-case  
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp1-backend:latest  
          push: ${{ github.ref == 'refs/heads/main' }}  
  
      - name: Build image and push database  
        uses: docker/build-push-action@v3  
        with:  
          # relative path to the place where source code with Dockerfile is located  
          context: ./TP1/Database  
          # Note: tags has to be all lower-case  
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp1-database:latest  
          push: ${{ github.ref == 'refs/heads/main' }}  
  
      - name: Build image and push httpd  
        uses: docker/build-push-action@v3  
        with:  
          # relative path to the place where source code with Dockerfile is located  
          context: ./TP1/HTTPServer  
          # Note: tags has to be all lower-case  
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp1-httpd:latest  
          push: ${{ github.ref == 'refs/heads/main' }}
```



