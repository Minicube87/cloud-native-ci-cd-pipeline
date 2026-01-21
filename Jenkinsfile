pipeline {
    agent any

    tools {
        maven 'Maven'
        nodejs 'Node'
    }

    stages {
        stage("Checkout") {
            steps {
                checkout scm
            }
        }

        stage("Build Backend") {
            steps {
                dir('backend') {
                    sh 'mvn clean package'
                }
            }
        }

        stage("Build Frontend") {
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm run build'
                    sh 'npm test'
                }
            }
        }

        stage("Docker Build Backend") {
            agent {
                docker {
                    image 'docker:24.0.7-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                dir('backend') {
                    sh 'docker build -t minicube78/kukuk-backend:latest .'
                }
            }
        }

        stage("Docker Push Backend") {
            agent {
                docker {
                    image 'docker:24.0.7-cli'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'docker push minicube78/kukuk-backend:latest'
            }
        }
    }

    post {
        always {
            cleanWs()
        }

        failure {
            echo "========pipeline execution failed========"
        }
    }
}
