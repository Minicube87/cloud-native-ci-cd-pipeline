pipeline{

    agent any

    tools{
      maven 'Maven'
      nodejs 'Node18'
    }

    stages{

        stage("Checkout"){
          steps{
            checkout scm
          }
        }
        
        stage("Build"){
          steps{
            dir('backend') {
              sh 'mvn clean package'
            }
          }
        }

        stage("Frontend"){
          steps{
            dir('frontend') {
              sh 'npm install'
              sh 'npm run build'
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
