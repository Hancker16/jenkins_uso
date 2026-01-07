pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  stages {
    stage('Checkout (manual)') {
      steps {
        deleteDir()
        checkout scm
      }
    }

    stage('Install + Build (Node)') {
      agent {
        docker { image 'node:20-alpine' }
      }
      steps {
        sh 'node -v'
        sh 'npm -v'
        sh 'npm install'
        sh 'npm run build'
      }
    }
  }
}
