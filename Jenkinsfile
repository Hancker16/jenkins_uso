pipeline {
  agent any

  stages {
    stage('Checkout') {
      steps {
        echo 'Clonando código'
      }
    }

    stage('Build') {
      steps {
        sh 'node -v'
        sh 'npm -v'
        sh 'npm run build'
      }
    }
  }
}
