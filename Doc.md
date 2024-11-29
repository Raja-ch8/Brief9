Contexte

L’objectif est de mettre en place un environnement permettant d’écrire du code de manière sécurisée et propre en utilisant des outils d’analyse tels que SonarQube ou SonarCloud. De plus, il est nécessaire d’intégrer des outils pour identifier les vulnérabilités des dépendances du projet, comme OWASP Dependency-Check.

L’une des premières difficultés rencontrées a été l’installation de Jenkins. Après plusieurs tentatives infructueuses, il a été conseillé de l’installer en utilisant Docker Compose et un Dockerfile.

Installation avec Docker Compose
Création du fichier docker-compose.yml :
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    ports:
      - "8080:8080"
    volumes:
      - jenkins_home:/var/jenkins_home
    restart: always
volumes:
  jenkins_home:
Création du fichier Dockerfile :
FROM jenkins/jenkins:lts
USER root 
ENV DEBIAN_FRONTEND=noninteractive
RUN apt update -y && \
    apt install -y jq parallel && \
    curl -sSL https://get.docker.com/ | sh
COPY config /root/.kube/config
👉 Remarque : La commande curl -sSL https://get.docker.com/ | sh installe automatiquement l’ensemble du package Docker.

Lancement du conteneur :
Exécuter la commande suivante :
docker-compose up -d
Cela démarre Jenkins en mode silencieux en masquant les journaux.
Pour obtenir le mot de passe Jenkins, exécuter :
docker logs jenkins
Intégration de OWASP Dependency-Check
Installer le plugin OWASP Dependency-Check via l’interface de Jenkins.
Configurer le plugin dans le pipeline Jenkins pour analyser les vulnérabilités des dépendances.
SonarQube et SonarCloud
Intégration de SonarCloud

Comme SonarQube nécessite un serveur dédié pour fonctionner, SonarCloud, la version cloud de SonarQube, a été choisie pour sa simplicité d’intégration avec Jenkins.

Installer le plugin SonarQube Scanner dans Jenkins.
Créer un jeton d’accès via le compte SonarCloud pour permettre l’authentification.
Pipeline Jenkins
Voici une configuration complète d’un pipeline Jenkins incluant OWASP Dependency-Check et SonarCloud :

pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker')
    }
    
    stages {
        stage('Nettoyage du Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Cloner le dépôt') {
            steps {
                sh 'git clone https://github.com/simplon-choukriraja/Brief8-Raja.git'
            }
        }
        
        stage('Construire l’image Docker') {
            steps {
                sh '''
                cd Brief8-Raja
                docker build -t vote-app .
                '''
            }
        }
        
        stage('Connexion à DockerHub') {
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
            }
        }
        
        stage('Pousser l’image') {
            steps {
                sh '''
                PATCH=$(cat Brief8-Raja/azure-vote/main.py | grep "ver = ".*"" | grep -oE "[0-9]+\\.[0-9]+\\.[0-9]+")
                docker tag vote-app raja8/vote-app:${PATCH}
                docker push raja8/vote-app:${PATCH}
                '''
            }
        }
        
        stage('Mise à jour de la configuration de l’application') {
            steps {
                sh '''
                git clone https://github.com/simplon-choukriraja/brief7-votinapp.git app
                TAG=$(curl -sSf https://registry.hub.docker.com/v2/repositories/raja8/vote-app/tags | jq '."results"[0]["name"]' | tr -d '"')
                sed -i "s/TAG/${TAG}/" ./app/vote.yml
                '''
            }
        }
        
        stage('Test de charge') {
            steps {
                sh '''
                seq 250 | parallel --max-args 0 --jobs 10 "curl -k -iF 'vote=Pizza' http://vote.simplon-raja.space"
                '''
            }
        }
        
        stage('Analyse OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '', odcInstallation: 'OWASP-Dependency-Check'
            }
        }
        
        stage('Publication des résultats') {
            steps {
                dependencyCheckPublisher pattern: ''
            }
        }
        
        stage('Analyse SonarCloud') {
            steps {
                withSonarQubeEnv('My SonarQube Server') {
                    sh 'mvn clean package sonar:sonar'
                }
            }
        }
        
        stage('Vérification Qualité') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }





















