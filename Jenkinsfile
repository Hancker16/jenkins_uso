pipeline {
  agent any
  tools {
    nodejs "Node_25"
  }

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
