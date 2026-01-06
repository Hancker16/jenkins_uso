pipeline {
  agent any

  tools {
    nodejs 'Node_25'
  }

  stages {
    stage('Build') {
      steps {
        bat 'node -v'
        bat 'npm -v'
        bat 'npm run build'
      }
    }
  }
}
