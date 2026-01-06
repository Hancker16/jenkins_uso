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
        sh 'npm run build'
      }
    }
  }
}
