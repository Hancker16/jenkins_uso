pipeline {
  agent any

  options {
    // Evita el checkout automático que te daba "fatal: not in a git directory"
    skipDefaultCheckout(true)
  }

  environment {
    APP_NAME  = "uso_jenkins"
    DOCKER_NET = "laboratio-ci_ci"

    // Nexus Docker Registry (puede requerir ajuste a host.docker.internal:8082 si localhost no responde desde Jenkins)
    REGISTRY = "localhost:8082"

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
        sh '''
          docker run --rm \
            -v "$PWD":/app -w /app \
            node:20-bookworm \
            bash -lc "npm install && npm run build"
        '''
      }
    }

    stage('SonarQube') {
      environment {
        SONAR_TOKEN = credentials('sonar-token')
      }
      steps {
        sh '''
          docker run --rm \
            --network laboratio-ci_ci \
            -v "$PWD":/app -w /app \
            node:20-bookworm \
            bash -lc "apt-get update && apt-get install -y openjdk-17-jre >/dev/null && \
                      npx --yes sonar-scanner \
                        -Dsonar.projectKey=uso_jenkins \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://sonarqube:9000 \
                        -Dsonar.login=$SONAR_TOKEN"
        '''
      }
    }

    stage('Docker Build Image') {
      steps {
        sh 'docker build -t "$IMAGE" .'
      }
    }

    stage('Push to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            echo "$NEXUS_PASS" | docker login "$REGISTRY" -u "$NEXUS_USER" --password-stdin
            docker push "$IMAGE"
          '''
        }
      }
    }
  }
}
