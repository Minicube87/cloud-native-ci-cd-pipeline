pipeline{

    agent any

    tools{
      maven 'Maven'
      nodejs 'Node'
    }

    stages{

        stage("Checkout"){
          steps{
            checkout scm
          }
        }
        
        stage("Build Backend"){
          steps{
            dir('backend') {
              sh 'mvn clean package'
            }
          }
        }

        stage("Build Frontend"){
          steps{
            dir('frontend') {
              sh 'npm install'
              sh 'npm run build'
              sh 'npm test'
            }
          }
        }

        stage("Docker Build Backend"){
          steps{
            dir('backend') {
              sh 'docker build -t kukuk-backend:latest .'
            }
          }
        }

       stage('Docker Push Backend') {
          steps {
            sh 'docker push minicube78/kukuk-backend:latest'
          }
        }
    }


    post{
        always{
            cleanWs()
        }

        failure{
            echo "========pipeline execution failed========"
        }
    }
}
