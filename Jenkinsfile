pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
    timestamps()
  }

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

    PULL_REGISTRY   = "host.docker.internal:8084"
    BASE_IMAGE_REPO = "library/node"

    INTERNET_REGISTRY = "docker.io"
    INTERNET_IMAGE    = "node"

    PUSH_REGISTRY = "host.docker.internal:8082"
    BASE_TAG = "${BUILD_NUMBER}"
  }

  stages {

    stage('Checkout') {
      steps {
        echo "[INFO] Checkout del repositorio y limpieza de workspace..."
        deleteDir()
        checkout scm
        sh 'rm -rf .scannerwork || true'
        echo "[OK] Código listo."
      }
    }

    stage('Resolve Project Image (local / nexus / internet)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            set -e

            log(){ echo "[INFO] $*"; }
            ok(){  echo "[OK]   $*"; }
            warn(){ echo "[WARN] $*"; }
            err(){ echo "[ERROR] $*"; }

            : > .ci_project_image
            echo "unset" > .ci_image_source

            NEXUS_IMG="${PULL_REGISTRY}/${BASE_IMAGE_REPO}:${BASE_IMAGE_TAG}"
            INTERNET_IMG="${INTERNET_IMAGE}:${BASE_IMAGE_TAG}"

            log "Resolviendo imagen base..."
            log " - Tag solicitado: ${BASE_IMAGE_TAG}"
            log " - Imagen objetivo (Nexus tag): ${NEXUS_IMG}"
            log " - Imagen alternativa (Internet): ${INTERNET_IMG}"

            # 1) Local
            if docker image inspect "$NEXUS_IMG" >/dev/null 2>&1; then
              echo "$NEXUS_IMG" > .ci_project_image
              echo "local" > .ci_image_source
              ok "Encontrada localmente => $NEXUS_IMG"
              exit 0
            fi

            # 2) Nexus
            log "No está local. Intentando descargar desde Nexus..."
            echo "$NEXUS_PASS" | docker login "$PULL_REGISTRY" -u "$NEXUS_USER" --password-stdin >/dev/null 2>&1 || true

            if docker pull "$NEXUS_IMG" >/dev/null 2>&1; then
              echo "$NEXUS_IMG" > .ci_project_image
              echo "nexus" > .ci_image_source
              ok "Descargada desde Nexus => $NEXUS_IMG"
              exit 0
            fi

            # 3) Internet
            warn "Nexus no tiene la imagen. Descargando desde Internet (Docker Hub)..."
            docker pull "$INTERNET_IMG" >/dev/null 2>&1
            docker tag "$INTERNET_IMG" "$NEXUS_IMG"

            echo "$NEXUS_IMG" > .ci_project_image
            echo "internet" > .ci_image_source
            ok "Descargada de Internet y retaggeada como Nexus => $NEXUS_IMG"
          '''
        }
      }
    }

    stage('Report Image Source') {
      steps {
        sh '''
          set -e
          IMG="$(cat .ci_project_image)"
          SRC="$(cat .ci_image_source)"
          echo "[INFO] Imagen base final: $IMG"
          echo "[INFO] Fuente: $SRC (local|nexus|internet)"
        '''
      }
    }

    stage('Build (npm)') {
      steps {
        sh '''
          set -e
          echo "[INFO] Build de la app (npm install + npm run build) usando contenedor Node..."
          PROJECT_IMAGE="$(cat .ci_project_image)"
          JENKINS_CID="$(hostname)"

          # Menos ruido: npm con logs reducidos
          docker run --rm \
            --volumes-from "$JENKINS_CID" \
            -w /var/jenkins_home/jobs/ci-cd-demo/workspace \
            "$PROJECT_IMAGE" \
            bash -lc "npm config set fund false >/dev/null 2>&1 || true; \
                      npm config set audit false >/dev/null 2>&1 || true; \
                      npm config set loglevel warn >/dev/null 2>&1 || true; \
                      npm install --silent; \
                      npm run build" 
          echo "[OK] Build npm completado."
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
          echo "[INFO] Ejecutando análisis SonarQube..."
          PROJECT_IMAGE="$(cat .ci_project_image)"
          JENKINS_CID="$(hostname)"

          # Silenciamos apt y dejamos sonar con output moderado
          docker run --rm \
            --network "$DOCKER_NET" \
            --volumes-from "$JENKINS_CID" \
            -w /var/jenkins_home/jobs/ci-cd-demo/workspace \
            "$PROJECT_IMAGE" \
            bash -lc "apt-get update -qq >/dev/null; \
                      DEBIAN_FRONTEND=noninteractive apt-get install -y -qq openjdk-17-jre >/dev/null; \
                      npx --yes sonar-scanner \
                        -Dsonar.projectKey=uso_jenkins \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://sonarqube:9000 \
                        -Dsonar.login=$SONAR_TOKEN" 

          echo "[OK] Scan enviado a SonarQube."
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

          info(){ echo "[INFO] $*"; }
          ok(){ echo "[OK]   $*"; }
          warn(){ echo "[WARN] $*"; }

          REPORT=".scannerwork/report-task.txt"
          if [ ! -f "$REPORT" ]; then
            warn "No existe $REPORT. Se marca como qg-f (fallo/indeterminado)."
            echo "qg-f" > .qg_tag
            exit 0
          fi

          CE_TASK_URL=$(grep -E '^ceTaskUrl=' "$REPORT" | cut -d= -f2-)
          info "Esperando resultado del análisis en Sonar (Compute Engine)..."

          ANALYSIS_ID=""
          STATUS=""

          for i in $(seq 1 120); do
            JSON=$(curl -s -u "$SONAR_TOKEN:" "$CE_TASK_URL")
            STATUS=$(echo "$JSON" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p' | head -n1)

            if [ "$STATUS" = "SUCCESS" ]; then
              ANALYSIS_ID=$(echo "$JSON" | sed -n 's/.*"analysisId":"\\([^"]*\\)".*/\\1/p' | head -n1)
              break
            fi

            if [ "$STATUS" = "FAILED" ] || [ "$STATUS" = "CANCELED" ]; then
              warn "Compute Engine terminó con status=$STATUS => qg-f"
              echo "qg-f" > .qg_tag
              exit 0
            fi

            sleep 2
          done

          if [ -z "$ANALYSIS_ID" ]; then
            warn "Timeout esperando analysisId => qg-f"
            echo "qg-f" > .qg_tag
            exit 0
          fi

          info "analysisId obtenido: $ANALYSIS_ID"
          QG_URL="http://sonarqube:9000/api/qualitygates/project_status?analysisId=$ANALYSIS_ID"
          QG_JSON=$(curl -s -u "$SONAR_TOKEN:" "$QG_URL")
          QG_STATUS=$(echo "$QG_JSON" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p' | head -n1)

          info "Quality Gate: $QG_STATUS"
          info "Detalle (API): $QG_URL"

          if [ "$QG_STATUS" = "OK" ]; then
            ok "Quality Gate PASSED => se etiquetará como qg-p"
            echo "qg-p" > .qg_tag
          else
            warn "Quality Gate FAILED => se etiquetará como qg-f"
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

          echo "[INFO] Construyendo imagen final de la app..."
          echo "[INFO] Tag final => $IMAGE"

          # build menos ruidoso
          docker build --quiet -t "$IMAGE" "${APP_DIR}" >/dev/null 2>&1 || docker build -t "$IMAGE" "${APP_DIR}"

          echo "$IMAGE" > .image_name
          echo "[OK] Imagen construida."
        '''
      }
    }

    stage('Push to Nexus (app image)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            set -e
            IMAGE=$(cat .image_name)

            echo "[INFO] Subiendo imagen final a Nexus..."
            echo "[INFO] Imagen => $IMAGE"

            echo "$NEXUS_PASS" | docker login "$PUSH_REGISTRY" -u "$NEXUS_USER" --password-stdin >/dev/null 2>&1 || true
            docker push "$IMAGE" >/dev/null 2>&1 || docker push "$IMAGE"

            echo "[OK] Imagen subida a Nexus => $IMAGE"
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

          echo "[INFO] Deploy de contenedor..."
          echo "[INFO] Contenedor => $CONTAINER_NAME"
          echo "[INFO] Imagen     => $IMAGE"

          docker stop "$CONTAINER_NAME" >/dev/null 2>&1 || true
          docker rm   "$CONTAINER_NAME" >/dev/null 2>&1 || true

          docker run -d --name "$CONTAINER_NAME" -p 3000:3000 "$IMAGE" >/dev/null
          echo "[OK] Deploy completado. URL: http://localhost:3000"
          docker ps --filter "name=$CONTAINER_NAME"
        '''
      }
    }

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

            info(){ echo "[INFO] $*"; }
            ok(){ echo "[OK]   $*"; }
            warn(){ echo "[WARN] $*"; }

            PROJECT_IMAGE="$(cat .ci_project_image)"
            info "Base image vino de LOCAL. Verificando si ya existe en Nexus..."
            info "Base image => $PROJECT_IMAGE"

            REPO_PATH="${BASE_IMAGE_REPO}"
            TAG="${BASE_IMAGE_TAG}"
            MANIFEST_URL="http://${PULL_REGISTRY}/v2/${REPO_PATH}/manifests/${TAG}"

            CODE=$(curl -sS -o /dev/null -w "%{http_code}" \
              -u "${NEXUS_USER}:${NEXUS_PASS}" \
              -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
              "$MANIFEST_URL" || true)

            if [ "$CODE" = "200" ]; then
              ok "Ya existe en Nexus (HTTP 200). No se publica."
              exit 0
            fi

            if [ "$CODE" = "404" ]; then
              warn "No existe en Nexus (HTTP 404). Publicando base image ahora..."
              echo "$NEXUS_PASS" | docker login "$PULL_REGISTRY" -u "$NEXUS_USER" --password-stdin >/dev/null 2>&1 || true
              docker push "$PROJECT_IMAGE" >/dev/null 2>&1 || docker push "$PROJECT_IMAGE"
              ok "Base image publicada => $PROJECT_IMAGE"
              exit 0
            fi

            warn "Código HTTP inesperado al consultar Nexus: $CODE. Por seguridad, no se publica."
          '''
        }
      }
    }

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

            echo "[INFO] Base image vino de INTERNET. Publicando directamente en Nexus..."
            echo "[INFO] Base image => $PROJECT_IMAGE"

            echo "$NEXUS_PASS" | docker login "$PULL_REGISTRY" -u "$NEXUS_USER" --password-stdin >/dev/null 2>&1 || true
            docker push "$PROJECT_IMAGE" >/dev/null 2>&1 || docker push "$PROJECT_IMAGE"

            echo "[OK] Base image publicada (internet) => $PROJECT_IMAGE"
          '''
        }
      }
    }

  }
}
