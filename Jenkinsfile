pipeline {
  agent any

  options {
    skipDefaultCheckout(true)
  }

  stages {
    stage('Checkout (manual)') {
      steps {
        deleteDir()        // limpia el workspace
        checkout scm       // clona el repo correctamente
      }
    }

    stage('Prueba') {
      steps {
        echo 'Checkout manual OK'
      }
    }
  }
}
