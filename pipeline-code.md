```groovy

pipeline {
    agent any
    tools {
        jdk 'jdk'
        nodejs 'nodejs'
    }
    environment {
        SCANNER_HOME = tool 'sonar'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', changelog: false, poll: false, url: 'https://github.com/Gaurav1251/Netflix-cicd'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        
        stage("Docker Build & Push"){
            steps{
                   
                   
                    withCredentials([usernamePassword(credentialsId: 'docker', passwordVariable: 'p', usernameVariable: 'u')]) {
                        sh "docker login -u ${env.u} -p ${env.p}"
                        sh "docker build --build-arg TMDB_V3_API_KEY=dcbb962f2e5980149bf0a562c396ff0b -t gaurav1251/netflix ."
                        
                        sh "docker push gaurav1251/netflix:latest "
    
                    }
                    
                
            }
        }
        
        stage("TRIVY"){
            steps{
                sh "trivy image gaurav1251/netflix:latest > trivyimage.txt" 
            }
        }
        
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 gaurav1251/netflix:latest'
            }
        }
    }
}
```
