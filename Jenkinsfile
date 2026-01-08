pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  environment {
    APP_NAME = "uso_jenkins"
    APP_DIR = "."
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
        sh 'rm -rf .scannerwork || true'

      }
    }

    stage('Build (npm)') {
      steps {
        sh '''
        JENKINS_CID = "$(hostname)"
        docker run--rm\
        --volumes - from "$JENKINS_CID"\ -
          w /
          var / jenkins_home / jobs / ci - cd - demo / workspace\
        node: 20 - bookworm\
        bash - lc "npm install && npm run build"
        '''
      }
    }

    stage('SonarQube Scan') {
      environment {
        SONAR_TOKEN = credentials('sonar-token')
      }
      steps {
        sh '''
        JENKINS_CID = "$(hostname)"
        docker run--rm\
        --network laboratio - ci_ci\
          --volumes - from "$JENKINS_CID"\ -
          w /
          var / jenkins_home / jobs / ci - cd - demo / workspace\
        node: 20 - bookworm\
        bash - lc "apt-get update && apt-get install -y openjdk-17-jre >/dev/null && \
                      npx --yes sonar-scanner \
                        -Dsonar.projectKey=uso_jenkins \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://sonarqube:9000 \
                        -Dsonar.login=$SONAR_TOKEN"
        '''
      }
    }

    stage('Quality Gate Result (no bloquea)') {
      environment {
        SONAR_TOKEN = credentials('sonar-token')
      }
      steps {
        sh '''
        set - e

        REPORT = ".scannerwork/report-task.txt"
        if [!-f "$REPORT"];
        then
        echo "No existe $REPORT => marcando qg-f"
        echo "qg-f" > .qg_tag
        exit 0
        fi

        echo "==== report-task.txt ===="
        cat "$REPORT"
        echo "========================="

        CE_TASK_URL = $(grep - E '^ceTaskUrl='
          "$REPORT" | cut - d = -f2 - )
        CE_TASK_ID = $(grep - E '^ceTaskId='
          "$REPORT" | cut - d = -f2 - )
        SERVER_URL = $(grep - E '^serverUrl='
          "$REPORT" | cut - d = -f2 - )

        echo "serverUrl=$SERVER_URL"
        echo "ceTaskId=$CE_TASK_ID"
        echo "ceTaskUrl=$CE_TASK_URL"

        ANALYSIS_ID = ""

        # esperar el procesamiento EXACTO de ese ceTaskUrl
        for i in $(seq 1 120);
        do
          JSON = $(curl - s - u "$SONAR_TOKEN:"
            "$CE_TASK_URL")
        STATUS = $(echo "$JSON" | sed - n 's/.*"status":"\\([^"]*\\)".*/\\1/p' | head - n1)

        if ["$STATUS" = "SUCCESS"];
        then
        ANALYSIS_ID = $(echo "$JSON" | sed - n 's/.*"analysisId":"\\([^"]*\\)".*/\\1/p' | head - n1)
        break
        fi

        if ["$STATUS" = "FAILED"] || ["$STATUS" = "CANCELED"];
        then
        echo "CE task falló status=$STATUS => qg-f"
        echo "qg-f" > .qg_tag
        exit 0
        fi

        sleep 2
        done

        if [-z "$ANALYSIS_ID"];
        then
        echo "Timeout esperando analysisId => qg-f"
        echo "qg-f" > .qg_tag
        exit 0
        fi

        echo "analysisId=$ANALYSIS_ID"

        # consultar el QG del MISMO analysisId
        QG_JSON = $(curl - s - u "$SONAR_TOKEN:"
          "http://sonarqube:9000/api/qualitygates/project_status?analysisId=$ANALYSIS_ID")
        QG_STATUS = $(echo "$QG_JSON" | sed - n 's/.*"status":"\\([^"]*\\)".*/\\1/p' | head - n1)

        echo "QG_JSON=$QG_JSON"
        echo "Quality Gate status: $QG_STATUS"

        if ["$QG_STATUS" = "OK"];
        then
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
        QG_TAG = $(cat.qg_tag)
        IMAGE = "${REGISTRY}/${APP_NAME}:${BASE_TAG}-${QG_TAG}"
        echo "Building image: $IMAGE"
        docker build - t "$IMAGE"
        "${APP_DIR}"
        echo "$IMAGE" > .image_name '''
      }
    }

    stage('Push to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-docker', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
          IMAGE = $(cat.image_name)
          echo "$NEXUS_PASS" | docker login "$REGISTRY" - u "$NEXUS_USER"--password - stdin
          docker push "$IMAGE"
          echo "Pushed: $IMAGE"
          '''
        }
      }
    }

    stage('Deploy docker') {
      steps {
        sh '''
        IMAGE = $(cat.image_name)
        CONTAINER_NAME = "uso_jenkins_app"

        echo "Deploy => image=${Image}"

        docker stop "$CONTAINER_NAME"
        2 > /dev/null || true
        docker rm "$CONTAINER_NAME"
        2 > /dev/null || true

        docker run - d--name "$CONTAINER_NAME" - p 3000: 3000 "$IMAGE"
        docker ps--filter "name=$CONTAINER_NAME"
        S
        echo "Deploy completado en http:localhost:3000"
        '''
      }
    }
  }
}