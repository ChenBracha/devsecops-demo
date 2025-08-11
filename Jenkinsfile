pipeline {
  agent any
  stages {
    stage('Sanity') {
      steps {
        echo "Jenkins can reach the repo and run steps"
        sh 'whoami && pwd && ls -la'
      }
    }
  }
}

