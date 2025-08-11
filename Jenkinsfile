pipeline {
  agent any

  environment {
    IMAGE_NAME   = "task-tracker:${BUILD_NUMBER}"
    CLUSTER_NAME = "devsecops-cluster"
    CHART_DIR    = "./task-tracker-chart"
    SMOKE_URL    = "http://localhost:8080/health"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '15'))
    ansiColor('xterm')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'echo "Git commit: $(git rev-parse --short HEAD)"'
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker version'          // proves CLI + socket
        sh 'docker build -t $IMAGE_NAME .'
        sh 'docker image ls | head -n 5'
      }
    }

    stage('Trivy Scan (non-blocking first)') {
      steps {
        sh 'trivy --version'
        // remove "|| true" later to gate on vulns
        sh 'trivy image --no-progress --severity HIGH,CRITICAL $IMAGE_NAME || true'
      }
    }

    stage('Import Image to k3d') {
      steps {
        sh 'k3d version'
        sh 'k3d image import $IMAGE_NAME -c $CLUSTER_NAME'
      }
    }

    stage('Helm Deploy') {
      steps {
        sh 'helm version'
        // upgrade/install chart; customize values if needed
        sh 'helm upgrade --install task-tracker $CHART_DIR'
        // optional: wait for rollout stability
        sh '''
          set -e
          APP=$(kubectl get deploy -A | awk '/task-tracker/ {print $2; exit}')
          [ -n "$APP" ] && kubectl rollout status deploy/$APP -n default --timeout=120s
        '''
      }
    }

    stage('Smoke Test') {
      steps {
        sh 'curl -sf $SMOKE_URL'
        echo "Smoke test OK: $SMOKE_URL"
      }
    }
  }

  post {
    success {
      echo "Deployed $IMAGE_NAME"
    }
    failure {
      echo "Build failed. Check the stage that broke (Docker/Trivy/k3d/Helm/Smoke)."
    }
    always {
      script {
        def dur = currentBuild.durationString
        echo "Build result: ${currentBuild.currentResult} in ${dur}"
      }
    }
  }
}