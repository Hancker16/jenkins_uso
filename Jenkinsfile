pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  environment {
    APP_NAME = "uso_jenkins"
    REGISTRY = "localhost:8082"  // Jenkins container publica hacia el host normalmente OK
    IMAGE = "${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        checkout scm
      }
    }

    stage('Build (npm)') {
      steps {
        // Ejecuta el build usando node en contenedor para tener npm disponible
        sh """
          docker run --rm \
            --network ${env.DOCKER_NET ?: ""} \
            -v "\$PWD":/app -w /app \
            node:20-bookworm \
            bash -lc "npm install && npm run build"
        """
      }
    }

    stage('SonarQube') {
      environment {
        SONAR_TOKEN = credentials('sonar-token')
      }
      steps {
        sh """
          docker run --rm \
            --network ${env.DOCKER_NET ?: ""} \
            -v "\$PWD":/app -w /app \
            node:20-bookworm \
            bash -lc "apt-get update && apt-get install -y openjdk-17-jre >/dev/null && \
                      npx --yes sonar-scanner \
                        -Dsonar.projectKey=${APP_NAME} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://sonarqube:9000 \
                        -Dsonar.login=$SONAR_TOKEN"
        """
      }
    }

    stage('Docker Build Image') {
      steps {
        sh "docker build -t ${IMAGE} ."
      }
    }

    stage('Push to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh """
            echo "$NEXUS_PASS" | docker login ${REGISTRY} -u "$NEXUS_USER" --password-stdin
            docker push ${IMAGE}
          """
        }
      }
    }
  }
}
