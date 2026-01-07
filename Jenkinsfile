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
        docker { image 'node:20-alpine' }pipeline {
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

      }
      steps {
        // Fuerza a usar el workspace principal donde sí está el repo
        dir("${env.WORKSPACE}") {
          sh 'ls -la'
          sh 'node -v'
          sh 'npm -v'
          sh 'npm install'
          sh 'npm run build'
        }
      }
    }
  }
}
