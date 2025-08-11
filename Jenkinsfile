pipeline {
  agent any

  environment {
    IMAGE_NAME   = "task-tracker:${BUILD_NUMBER}"
    CLUSTER_NAME = "devsecops-cluster"
    CHART_DIR    = "./task-tracker-chart"
    NAMESPACE    = "default"
    SMOKE_URL    = "http://localhost:8080/health"
    // kubectl/helm will look here; we mount it from host in docker run
    KUBECONFIG   = "/var/jenkins_home/.kube/config"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm   // avoids the earlier 'master vs main' issue
        sh 'echo "Git commit: $(git rev-parse --short HEAD)"'
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker version'
        sh 'docker build -t $IMAGE_NAME .'
        sh 'docker image ls | head -n 5'
      }
    }

    stage('Trivy Scan (non-blocking)') {
      steps {
        sh 'trivy --version'
        // remove "|| true" later to gate the build on HIGH/CRITICAL vulns
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
        sh 'helm version && kubectl version --client'
        sh '''
          set -e
          kubectl get ns "$NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$NAMESPACE"
          helm upgrade --install task-tracker "$CHART_DIR" -n "$NAMESPACE"
          kubectl rollout status deploy/task-tracker -n "$NAMESPACE" --timeout=120s || true
        '''
      }
    }

    stage('Smoke Test') {
      steps {
        // try for up to ~30s while pods come up
        sh '''
          set -e
          for i in $(seq 1 15); do
            if curl -fsS "$SMOKE_URL" >/dev/null; then
              echo "Smoke OK: $SMOKE_URL"
              exit 0
            fi
            echo "Waiting for app... ($i/15)"; sleep 2
          done
          echo "Smoke test failed: $SMOKE_URL"; exit 1
        '''
      }
    }
  }

  post {
    success { echo "Deployed $IMAGE_NAME to $CLUSTER_NAME ($NAMESPACE)" }
    failure { echo "Build failed — check the failing stage’s logs." }
    always  { echo "Build result: ${currentBuild.currentResult}" }
  }
}
