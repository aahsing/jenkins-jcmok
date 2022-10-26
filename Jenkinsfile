pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/aahsing/jenkins-jcmok.git']]])
                sh 'echo "Hello"'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("jclahoho/nginx-test")
                }

            }
        }
    }
}