pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  stages {
    stage('Node: Checkout + Install + Build + Sonar') {
      agent {
        docker { image 'node:20-alpine' }
      }
      environment {
        SONAR_TOKEN = credentials('sonar-token')
      }
      steps {
        deleteDir()
        checkout scm

        sh 'npm install'
        sh 'npm run build'

        // Instala y ejecuta el scanner
        sh 'npx --yes sonar-scanner -v'

        sh """
          npx --yes sonar-scanner \
            -Dsonar.projectKey=uso_jenkins \
            -Dsonar.sources=. \
            -Dsonar.host.url=http://sonarqube:9000 \
            -Dsonar.login=$SONAR_TOKEN
        """
      }
    }
  }
}
