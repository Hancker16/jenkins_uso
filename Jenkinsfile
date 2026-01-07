pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  environment {
    APP_NAME   = "uso_jenkins"
    APP_DIR    = "."               // <-- si tu package.json está en una subcarpeta, ponla aquí (ej: "app" o "uso_jenkins")
    DOCKER_NET = "laboratio-ci_ci"

    REGISTRY = "localhost:8082"
    IMAGE    = "${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        checkout scm
        sh 'pwd'
        sh 'ls -la'
        sh 'ls -la "${APP_DIR}"'
      }
    }

    stage('Build (npm)') {
      steps {
        sh '''
          JENKINS_CID="$(hostname)"
          docker run --rm \
            --volumes-from "$JENKINS_CID" \
            -w /var/jenkins_home/jobs/ci-cd-demo/workspace \
            node:20-bookworm \
            bash -lc "ls -la && npm install && npm run build"
        '''
      }
    }


    stage('SonarQube') {
      environment {
        SONAR_TOKEN = credentials('sonar-token')
      }
      steps {
        sh '''
          JENKINS_CID="$(hostname)"
          docker run --rm \
            --network laboratio-ci_ci \
            --volumes-from "$JENKINS_CID" \
            -w /var/jenkins_home/jobs/ci-cd-demo/workspace \
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
        sh '''
          docker build -t "$IMAGE" "${APP_DIR}"
        '''
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
