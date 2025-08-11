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
        // Run Trivy but do NOT auto-fail; we’ll decide based on parsed counts.
          sh """
          set -e
          trivy image --severity HIGH,CRITICAL --no-progress ${IMAGE} | tee trivy_output.txt
           """

      // Pull accurate counts from the "Total:" line
      def parts = sh(script: '''
        LINE=$(grep -E "^Total:" trivy_output.txt | tail -n1)
        TOTAL=$(echo "$LINE"   | sed -n 's/^Total:\\s*\\([0-9]\\+\\).*/\\1/p')
        HIGH=$(echo "$LINE"    | sed -n 's/.*HIGH:\\s*\\([0-9]\\+\\).*/\\1/p')
        CRIT=$(echo "$LINE"    | sed -n 's/.*CRITICAL:\\s*\\([0-9]\\+\\).*/\\1/p')
        echo "$TOTAL $HIGH $CRIT"
      ''', returnStdout: true).trim().split(' ')

        def total = parts[0] as int
        def high  = parts[1] as int
        def crit  = parts[2] as int

         echo "🔍 Trivy Summary: ${total} total — ${high} HIGH, ${crit} CRITICAL."

         if (high > 0 || crit > 0) {
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
