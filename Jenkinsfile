pipeline {
  agent any

  environment {
    IMAGE_NAME   = "task-tracker:${BUILD_NUMBER}"
    CLUSTER_NAME = "devsecops-cluster"
    CHART_DIR    = "./task-tracker-chart"
    NAMESPACE    = "default"
    SMOKE_URL    = "http://localhost:8080/health"
    // we'll create this file in the 'Fix kubeconfig' stage:
    KUBECONFIG   = "/var/jenkins_home/kube/config"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '15'))
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
        sh 'docker version'
        sh 'docker build -t $IMAGE_NAME .'
        sh 'docker image ls | head -n 5'
      }
    }

    stage('Trivy Scan (non-blocking)') {
      steps {
        sh 'trivy --version'
        // keep non-blocking while you iterate; later remove "|| true"
        sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL --no-progress task-tracker:${IMAGE_TAG}'
      }
    }

    stage('Import Image to k3d') {
      steps {
        sh 'k3d version'
        sh 'k3d image import $IMAGE_NAME -c $CLUSTER_NAME'
      }
    }

    // ---- IMPORTANT: make kubeconfig valid *inside* the container ----
    stage('Fix kubeconfig for Jenkins container') {
      steps {
        sh '''
          set -eux
          mkdir -p /var/jenkins_home/kube

          # Use mounted kubeconfig if present; else generate from k3d
          if [ -f /var/jenkins_home/.kube/config ]; then
            cp /var/jenkins_home/.kube/config "$KUBECONFIG"
          else
            k3d kubeconfig get "${CLUSTER_NAME}" > "$KUBECONFIG"
          fi

          # Replace server to the k3d LB hostname that matches cluster certs
          # Cert SAN includes: k3d-<cluster>-serverlb, localhost, etc.
          # Use internal LB name so TLS verifies OK.
          sed -E "s#https://[^:]+:[0-9]+#https://k3d-${CLUSTER_NAME}-serverlb:6443#g" -i "$KUBECONFIG"

          echo "Using KUBECONFIG=$KUBECONFIG"
          KUBECONFIG="$KUBECONFIG" kubectl cluster-info
        '''
        script {
          // Make KUBECONFIG visible to the following stages
          env.KUBECONFIG = '/var/jenkins_home/kube/config'
        }
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
    success { echo "✅ Deployed $IMAGE_NAME to $CLUSTER_NAME ($NAMESPACE)" }
    failure { echo "❌ Build failed — see the stage logs above." }
    always  { echo "Build result: ${currentBuild.currentResult}" }
  }
}
