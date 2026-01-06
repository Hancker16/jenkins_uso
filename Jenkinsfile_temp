pipeline {
  agent any
  stages {
    stage('Info') {
      steps {
        echo "Branch: ${env.BRANCH_NAME}"
        echo "Change ID (PR): ${env.CHANGE_ID}"
        sh 'git log -1 --oneline || true'
      }
    }
  }
}
