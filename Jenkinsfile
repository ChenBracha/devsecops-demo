pipeline {
  agent any
  environment {
    APP        = 'task-tracker'
    KUBECONFIG = '/var/jenkins_home/.kube/config'   // already mounted/copied
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          env.IMAGE = "${APP}:${TAG}"
          echo "Building ${IMAGE}"
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker build -t ${IMAGE} .'
        sh 'docker image ls | head -n 5'
      }
    }

   stage('Trivy Scan (blocking)') {
    steps {
        script {
            def trivyResult = sh(script: "trivy image --severity HIGH,CRITICAL --no-progress ${IMAGE} | tee trivy_output.txt", returnStatus: true)

            // Parse the counts from Trivy output
            def highCount = sh(script: "grep -c 'HIGH' trivy_output.txt || true", returnStdout: true).trim()
            def criticalCount = sh(script: "grep -c 'CRITICAL' trivy_output.txt || true", returnStdout: true).trim()

            echo "🔍 Trivy Summary: ${highCount} HIGH, ${criticalCount} CRITICAL vulnerabilities found."

            // Fail the build if vulnerabilities found
            if (highCount.toInteger() > 0 || criticalCount.toInteger() > 0) {
                error "❌ Pipeline failed due to security vulnerabilities."
            }
        }
      }
    }

    stage('Import Image to k3d') {
      steps {
        sh 'k3d image import ${IMAGE} -c devsecops-cluster'
      }
    }

    stage('Helm Deploy') {
      steps {
        sh '''
          set -e
          kubectl get ns default
          helm upgrade --install task-tracker ./task-tracker-chart \
            -n default \
            --set image.repository=${APP} \
            --set image.tag=${TAG}
          kubectl rollout status deploy/task-tracker -n default --timeout=120s
        '''
      }
    }

    stage('Smoke Test') {
      steps {
        sh 'curl -fsS http://localhost:8080/health >/dev/null && echo "Smoke OK"'
      }
    }
  }

  post {
    success { echo "✅ Deployed ${IMAGE} to devsecops-cluster (default)" }
    failure { echo "❌ Build failed — check the failing stage’s logs." }
  }
}
