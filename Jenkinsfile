pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  stages {
    stage('Node: Checkout + Install + Build') {
      agent {
        docker { image 'node:20-alpine' }
      }
      steps {
        deleteDir()
        checkout scm

        sh 'ls -la'
        sh 'node -v'
        sh 'npm -v'
        sh 'npm install'
        sh 'npm run build'
      }
    }
  }
}
