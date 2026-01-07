pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }
stages {
stage('Node: Checkout + Install + Build + Sonar') {
  agent {
    docker {
      image 'node:20-bookworm'
      args '--network ci'
    }
  environment {
    SONAR_TOKEN = credentials('sonar-token')
  }
  steps {
    deleteDir()
    checkout scm

    sh 'npm install'
    sh 'npm run build'

    // Java para sonar-scanner
    sh 'apt-get update && apt-get install -y openjdk-17-jre'
    sh 'java -version'

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
