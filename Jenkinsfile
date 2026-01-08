pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  // Tag variable para la imagen base (puedes cambiarlo en cada build)
  parameters {
    string(name: 'BASE_IMAGE_TAG', defaultValue: '20-bookworm', description: 'Tag de la imagen base (ej: 20-bookworm, 20-alpine, 22-bookworm)')
  }

  environment {
    APP_NAME   = "uso_jenkins"
    APP_DIR    = "."
    DOCKER_NET = "laboratio-ci_ci"

    // Nexus para BAJAR base images (Node). Jenkins corre en contenedor
    PULL_REGISTRY   = "host.docker.internal:8084"
    BASE_IMAGE_REPO = "library/node"

    // Imagen deseada del proyecto (variable por tag)
    DESIRED_PROJECT_IMAGE = "${PULL_REGISTRY}/${BASE_IMAGE_REPO}:${params.BASE_IMAGE_TAG}"

    // Nexus para SUBIR tu imagen final
    PUSH_REGISTRY = "host.docker.internal:8082"

    // Tag base para tu imagen final
    BASE_TAG = "${BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        deleteDir()
        checkout scm
        sh 'rm -rf .scannerwork || true'
      }
    }

    stage('Resolve Project Image (local or nexus)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            set -e

            # Inicializar "variables" persistidas en workspace
            : > .ci_project_image
            echo "unset" > .ci_image_source

            IMG="${DESIRED_PROJECT_IMAGE}"
            echo "Desired base image: $IMG"

            # Step 1: si está local, úsala
            if docker image inspect "$IMG" >/dev/null 2>&1; then
              echo "$IMG" > .ci_project_image
              echo "local" > .ci_image_source
              echo "[resolver] Found locally => $IMG"
              exit 0
            fi

            # Step 2: si no está local, bajar de Nexus
            echo "[resolver] Not found locally. Pulling from Nexus..."
            echo "$NEXUS_PASS" | docker login "$PULL_REGISTRY" -u "$NEXUS_USER" --password-stdin
            docker pull "$IMG"

            echo "$IMG" > .ci_project_image
            echo "nexus" > .ci_image_source
            echo "[resolver] Pulled from Nexus => $IMG"
          '''
        }
      }
    }

    stage('Report Image Source') {
      steps {
        sh '''
          set -e
          echo "PROJECT_IMAGE=$(cat .ci_project_image)"
          echo "IMAGE_SOURCE=$(cat .ci_image_source)"
        '''
      }
    }

    stage('Build (npm)') {
      steps {
        sh '''
          set -e
          PROJECT_IMAGE="$(cat .ci_project_image)"
          JENKINS_CID="$(hostname)"

          docker run --rm \
            --volumes-from "$JENKINS_CID" \
            -w /var/jenkins_home/jobs/ci-cd-demo/workspace \
            "$PROJECT_IMAGE" \
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
          set -e
          PROJECT_IMAGE="$(cat .ci_project_image)"
          JENKINS_CID="$(hostname)"

          docker run --rm \
            --network "$DOCKER_NET" \
            --volumes-from "$JENKINS_CID" \
            -w /var/jenkins_home/jobs/ci-cd-demo/workspace \
            "$PROJECT_IMAGE" \
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
          set -e

          REPORT=".scannerwork/report-task.txt"
          if [ ! -f "$REPORT" ]; then
            echo "No existe $REPORT => marcando qg-f"
            echo "qg-f" > .qg_tag
            exit 0
          fi

          echo "==== report-task.txt ===="
          cat "$REPORT"
          echo "========================="

          CE_TASK_URL=$(grep -E '^ceTaskUrl=' "$REPORT" | cut -d= -f2-)

          ANALYSIS_ID=""

          # esperar el procesamiento EXACTO de ese ceTaskUrl
          for i in $(seq 1 120); do
            JSON=$(curl -s -u "$SONAR_TOKEN:" "$CE_TASK_URL")
            STATUS=$(echo "$JSON" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p' | head -n1)

            if [ "$STATUS" = "SUCCESS" ]; then
              ANALYSIS_ID=$(echo "$JSON" | sed -n 's/.*"analysisId":"\\([^"]*\\)".*/\\1/p' | head -n1)
              break
            fi

            if [ "$STATUS" = "FAILED" ] || [ "$STATUS" = "CANCELED" ]; then
              echo "CE task falló status=$STATUS => qg-f"
              echo "qg-f" > .qg_tag
              exit 0
            fi

            sleep 2
          done

          if [ -z "$ANALYSIS_ID" ]; then
            echo "Timeout esperando analysisId => qg-f"
            echo "qg-f" > .qg_tag
            exit 0
          fi

          echo "analysisId=$ANALYSIS_ID"

          # consultar el QG del MISMO analysisId
          QG_JSON=$(curl -s -u "$SONAR_TOKEN:" "http://sonarqube:9000/api/qualitygates/project_status?analysisId=$ANALYSIS_ID")
          QG_STATUS=$(echo "$QG_JSON" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p' | head -n1)

          echo "QG_JSON=$QG_JSON"
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
          set -e
          QG_TAG=$(cat .qg_tag)
          IMAGE="${PUSH_REGISTRY}/${APP_NAME}:${BASE_TAG}-${QG_TAG}"
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
            set -e
            IMAGE=$(cat .image_name)
            echo "$NEXUS_PASS" | docker login "$PUSH_REGISTRY" -u "$NEXUS_USER" --password-stdin
            docker push "$IMAGE"
            echo "Pushed: $IMAGE"
          '''
        }
      }
    }

    stage('Deploy docker') {
      steps {
        sh '''
          set -e
          IMAGE=$(cat .image_name)
          CONTAINER_NAME="uso_jenkins_app"

          echo "Deploy => image=$IMAGE"

          docker stop "$CONTAINER_NAME" 2>/dev/null || true
          docker rm "$CONTAINER_NAME" 2>/dev/null || true

          docker run -d --name "$CONTAINER_NAME" -p 3000:3000 "$IMAGE"
          docker ps --filter "name=$CONTAINER_NAME"
          echo "Deploy completado en http://localhost:3000"
        '''
      }
    }
  }
}
