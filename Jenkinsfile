pipeline {
  agent any

  options {
    // Evita el checkout automático que te estaba rompiendo el workspace
    skipDefaultCheckout(true)
  }

  environment {
    APP_NAME = "uso_jenkins"
    SONAR_HOST = "http://sonarqube:9000"
    SONAR_PROJECT_KEY = "uso_jenkins"
  }

  stages {
    stage('Node: Checkout + Install + Build + Sonar') {
      agent {
        docker {
          image 'node:20-bookworm'
          // Importante: une el contenedor temporal a la misma red docker del compose
          args '--network laboratio-ci_ci'
        }
      }

      environment {
        SONAR_TOKEN = credentials('sonar-token')
      }

      steps {
        // Checkout dentro del mismo contexto (workspace) donde corre npm/sonar
        deleteDir()
        checkout scm

        // Build
        sh 'node -v'
        sh 'npm -v'
        sh 'npm install'
        sh 'npm run build'

        // Sonar Scanner necesita Java
        sh 'apt-get update && apt-get install -y openjdk-17-jre'
        sh 'java -version'

        // Ejecutar análisis en SonarQube
        sh """
          npx --yes sonar-scanner \
            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
            -Dsonar.sources=. \
            -Dsonar.host.url=${SONAR_HOST} \
            -Dsonar.login=${SONAR_TOKEN}
        """
      }
    }
  }
}
