pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS=credentials('docker')
    }
    
    stages{
        stage('CleanWs'){
            // Nettoyage de l'espace de travail Jenkins 
            steps{
            cleanWs()
                
            }
        }
    
            
        stage('Clone Repository') {
            steps{
                sh('''
                git clone https://github.com/simplon-choukriraja/Brief8-Raja.git
                ''')
            }
        }
        
        
        stage('Build Image') {
            steps {
                sh ('''
                cd Brief8-Raja
                docker build -t vote-app .
                ''')
            }
        }
        
        
        stage('Login') {
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                
            }
        }
        
        
        stage('Push') {
            steps {
                sh ('''
                PATCH=$(cat Brief8-Raja/azure-vote/main.py | grep "ver = ".*"" | grep -oE "[0-9]+\\.[0-9]+\\.[0-9]+")
                docker tag vote-app raja8/vote-app:\044PATCH
                docker push raja8/vote-app:\044PATCH
    
                ''')
            }
        }
        
        stage('Clone repository app-vote') {
            steps {
                    sh('''
                    git clone https://github.com/simplon-choukriraja/brief7-votinapp.git app
                    TAG=\044(curl -sSf https://registry.hub.docker.com/v2/repositories/raja8/vote-app/tags |jq '."results"[0]["name"]'| tr -d '"')
                    sed -i "s/TAG/\044{TAG}/" ./app/vote.yml
                    ''')
                
            }
        }
        
        stage('Test de charge'){
            steps{
                sh('''
                seq 250 | parallel --max-args 0  --jobs 10 "curl -k -iF 'vote=Pizza' http://vote.simplon-raja.space"
                ''')
            }
        }
        
        stage('Owasp'){
            steps{
                dependencyCheck additionalArguments: '', odcInstallation: 'OWASP-Dependency-Check'
            }
        }
        
        stage('Result'){
            steps{
                dependencyCheckPublisher pattern: ''
                
            }
        }
        stage("build & SonarQube analysis") {
            steps {
              withSonarQubeEnv('My SonarQube Server') {
                sh 'mvn clean package sonar:sonar'
              }
            }
        }
          
            
        stage("Quality Gate") {
            steps {
              timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true
              }
            }
        }
    }
    
}
