pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  // Tag variable para la imagen base (puedes cambiarlo en cada build)
  parameters {
    string(
      name: 'BASE_IMAGE_TAG',
      defaultValue: '20-alpine',
      description: 'Tag de la imagen base (ej: 20-bookworm, 20-alpine, 22-bookworm)'
    )
  }

  environment {
    APP_NAME   = "uso_jenkins"
    APP_DIR    = "."
    DOCKER_NET = "laboratio-ci_ci"

    // Nexus para BAJAR/ALMACENAR base images (Node)
    PULL_REGISTRY   = "host.docker.internal:8084"
    BASE_IMAGE_REPO = "library/node"

    // Internet/DockerHub base (misma repo, distinto registry)
    INTERNET_REGISTRY = "docker.io"
    INTERNET_IMAGE    = "node"

    // Nexus para SUBIR tu imagen final (tu app)
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

    // Resolver imagen base:
    // - Si está en local => IMAGE_SOURCE=local
    // - Si no está en local:
    //    - intenta Nexus (pull)
    //    - si no existe en Nexus => baja de internet (docker hub) y la retaggea a Nexus => IMAGE_SOURCE=internet
    stage('Resolve Project Image (local / nexus / internet)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            set -e

            : > .ci_project_image
            echo "unset" > .ci_image_source

            # Imagen destino (la que SIEMPRE usará el pipeline)
            NEXUS_IMG="${PULL_REGISTRY}/${BASE_IMAGE_REPO}:${BASE_IMAGE_TAG}"

            # Imagen origen internet (docker hub)
            INTERNET_IMG="${INTERNET_IMAGE}:${BASE_IMAGE_TAG}"

            echo "BASE_IMAGE_TAG=$BASE_IMAGE_TAG"
            echo "Desired (Nexus tag): $NEXUS_IMG"
            echo "Internet source:     $INTERNET_IMG"

            # Step 1: si existe localmente el tag de Nexus, úsalo
            if docker image inspect "$NEXUS_IMG" >/dev/null 2>&1; then
              echo "$NEXUS_IMG" > .ci_project_image
              echo "local" > .ci_image_source
              echo "[resolver] Found locally => $NEXUS_IMG"
              exit 0
            fi

            # Step 2: intentar bajar desde Nexus
            echo "[resolver] Not found locally. Trying Nexus pull..."
            echo "$NEXUS_PASS" | docker login "$PULL_REGISTRY" -u "$NEXUS_USER" --password-stdin

            if docker pull "$NEXUS_IMG"; then
              echo "$NEXUS_IMG" > .ci_project_image
              echo "nexus" > .ci_image_source
              echo "[resolver] Pulled from Nexus => $NEXUS_IMG"
              exit 0
            fi

            # Step 3: si Nexus no lo tiene, bajar de internet (Docker Hub) y retaggear al nombre Nexus
            echo "[resolver] Nexus doesn't have it. Pulling from Internet (Docker Hub)..."
            docker pull "$INTERNET_IMG"

            # Retag a nombre Nexus para que el resto del pipeline use el mismo nombre siempre
            docker tag "$INTERNET_IMG" "$NEXUS_IMG"

            echo "$NEXUS_IMG" > .ci_project_image
            echo "internet" > .ci_image_source
            echo "[resolver] Pulled from Internet and retagged => $NEXUS_IMG"
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

    stage('Push to Nexus (app image)') {
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

    // ✅ Si la imagen se tomó de LOCAL, verifica si existe en Nexus; si no existe, la sube.
    stage('Publish Base Image to Nexus (only if local & missing)') {
      when {
        expression {
          return fileExists('.ci_image_source') && readFile('.ci_image_source').trim() == 'local'
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            set -e

            PROJECT_IMAGE="$(cat .ci_project_image)"
            echo "[base-publish] IMAGE_SOURCE=local => checking in Nexus..."
            echo "[base-publish] PROJECT_IMAGE=$PROJECT_IMAGE"

            REPO_PATH="${BASE_IMAGE_REPO}"
            TAG="${BASE_IMAGE_TAG}"
            MANIFEST_URL="http://${PULL_REGISTRY}/v2/${REPO_PATH}/manifests/${TAG}"

            echo "[base-publish] Checking: $MANIFEST_URL"

            CODE=$(curl -sS -o /dev/null -w "%{http_code}" \
              -u "${NEXUS_USER}:${NEXUS_PASS}" \
              -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
              "$MANIFEST_URL" || true)

            if [ "$CODE" = "200" ]; then
              echo "[base-publish] Base image already exists in Nexus (HTTP 200). Doing nothing."
              exit 0
            fi

            if [ "$CODE" = "404" ]; then
              echo "[base-publish] Base image NOT found in Nexus (HTTP 404). Pushing now..."
              echo "$NEXUS_PASS" | docker login "$PULL_REGISTRY" -u "$NEXUS_USER" --password-stdin
              docker push "$PROJECT_IMAGE"
              echo "[base-publish] Pushed base image => $PROJECT_IMAGE"
              exit 0
            fi

            echo "[base-publish] Unexpected HTTP code from Nexus: $CODE"
            echo "[base-publish] For safety, not pushing."
          '''
        }
      }
    }

    // ✅ Si la imagen se tomó de INTERNET, después del deploy se sube DIRECTO a Nexus (sin consultar).
    stage('Publish Base Image to Nexus (direct if internet)') {
      when {
        expression {
          return fileExists('.ci_image_source') && readFile('.ci_image_source').trim() == 'internet'
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            set -e
            PROJECT_IMAGE="$(cat .ci_project_image)"

            echo "[base-publish] IMAGE_SOURCE=internet => pushing directly to Nexus..."
            echo "[base-publish] PROJECT_IMAGE=$PROJECT_IMAGE"

            echo "$NEXUS_PASS" | docker login "$PULL_REGISTRY" -u "$NEXUS_USER" --password-stdin
            docker push "$PROJECT_IMAGE"

            echo "[base-publish] Pushed base image (internet) => $PROJECT_IMAGE"
          '''
        }
      }
    }

  }
}
