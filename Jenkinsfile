pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  environment {
    APP_NAME   = "uso_jenkins"
    APP_DIR    = "."
    DOCKER_NET = "laboratio-ci_ci"

    // Tu registry en Nexus (desde Jenkins en Docker Desktop)
    REGISTRY = "host.docker.internal:8082"

    // Tag base
    BASE_TAG = "${BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        checkout scm
        sh 'ls -la'
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
            bash -lc "npm install && npm run build"
        '''
      }
    }

    stage('SonarQube Scan') {
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

    stage('Quality Gate Result') {
      environment {
        SONAR_TOKEN = credentials('sonar-token')
      }
      steps {
        sh '''
          set +e

          # Lee el task url del scanner
          REPORT=".scannerwork/report-task.txt"
          CE_TASK_URL=$(grep -E '^ceTaskUrl=' "$REPORT" | cut -d= -f2-)

          # Espera a que Sonar termine el procesamiento
          ANALYSIS_ID=""
          for i in $(seq 1 90); do
            JSON=$(curl -s -u "$SONAR_TOKEN:" "$CE_TASK_URL")
            STATUS=$(echo "$JSON" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p' | head -n1)

            if [ "$STATUS" = "SUCCESS" ]; then
              ANALYSIS_ID=$(echo "$JSON" | sed -n 's/.*"analysisId":"\\([^"]*\\)".*/\\1/p' | head -n1)
              break
            fi
            sleep 2
          done

          if [ -z "$ANALYSIS_ID" ]; then
            echo "qg-f" > .qg_tag
            echo "No se pudo obtener analysisId => marcando como qg-f"
            exit 0
          fi

          # Consulta el QG
          QG_JSON=$(curl -s -u "$SONAR_TOKEN:" "http://sonarqube:9000/api/qualitygates/project_status?analysisId=$ANALYSIS_ID")
          QG_STATUS=$(echo "$QG_JSON" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p' | head -n1)

          echo "Quality Gate status: $QG_STATUS"

          if [ "$QG_STATUS" = "OK" ]; then
            echo "qg-p" > .qg_tag
          else
            echo "qg-f" > .qg_tag
          fi
        '''
      }
    }

    stage('Docker Build Image') {
      steps {
        sh '''
          QG_TAG=$(cat .qg_tag)
          IMAGE="${REGISTRY}/${APP_NAME}:${BASE_TAG}-${QG_TAG}"
          echo "Building image: $IMAGE"
          docker build -t "$IMAGE" "${APP_DIR}"
          echo "$IMAGE" > .image_name
        '''
      }
    }

    stage('Push to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            IMAGE=$(cat .image_name)
            echo "$NEXUS_PASS" | docker login "$REGISTRY" -u "$NEXUS_USER" --password-stdin
            docker push "$IMAGE"
            echo "Pushed: $IMAGE"
          '''
        }
      }
    }
  }
}
