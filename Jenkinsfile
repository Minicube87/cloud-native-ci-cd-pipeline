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
