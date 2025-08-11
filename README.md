# --- FILE: Dockerfile.jenkins
# Jenkins controller image with Docker CLI, Trivy, kubectl, Helm, k3d

FROM jenkins/jenkins:lts

USER root

# Basic deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl gnupg git jq apt-transport-https lsb-release \
    && rm -rf /var/lib/apt/lists/*

# -----------------------------
# Docker CLI (client only)
# -----------------------------
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo ${VERSION_CODENAME}) stable" > /etc/apt/sources.list.d/docker.list \
 && apt-get update && apt-get install -y docker-ce-cli \
 && rm -rf /var/lib/apt/lists/*

# -----------------------------
# Trivy
# -----------------------------
RUN curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor -o /usr/share/keyrings/trivy.gpg \
 && echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb stable main" > /etc/apt/sources.list.d/trivy.list \
 && apt-get update && apt-get install -y trivy \
 && rm -rf /var/lib/apt/lists/*

# -----------------------------
# kubectl
# -----------------------------
ARG KUBECTL_VERSION=v1.31.0
RUN curl -fsSL -o /usr/local/bin/kubectl https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl \
 && chmod +x /usr/local/bin/kubectl

# -----------------------------
# Helm
# -----------------------------
ARG HELM_VERSION=v3.14.4
RUN curl -fsSL https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz -o /tmp/helm.tgz \
 && tar -xzf /tmp/helm.tgz -C /tmp \
 && mv /tmp/linux-amd64/helm /usr/local/bin/helm \
 && chmod +x /usr/local/bin/helm \
 && rm -rf /tmp/helm.tgz /tmp/linux-amd64

# -----------------------------
# k3d
# -----------------------------
ARG K3D_VERSION=v5.8.3
RUN curl -fsSL https://github.com/k3d-io/k3d/releases/download/${K3D_VERSION}/k3d-linux-amd64 -o /usr/local/bin/k3d \
 && chmod +x /usr/local/bin/k3d

# Let Jenkins user access docker socket (works when socket is mounted)
RUN groupadd -g 999 docker || true \
 && usermod -aG docker jenkins

USER jenkins

# Default Jenkins ports are already exposed by base image
# Run with:
# docker run -d --name jenkins \
#   -p 8080:8080 -p 50000:50000 \
#   -v jenkins_home:/var/jenkins_home \
#   -v /var/run/docker.sock:/var/run/docker.sock \
#   -v $HOME/.kube:/var/jenkins_home/.kube \
#   chenbracha/jenkins-devsecops:latest


# --- FILE: Jenkinsfile
pipeline {
  agent any

  options {
    timestamps()
  }

  environment {
    APP_NAME        = 'task-tracker'
    IMAGE           = ''                                // will be set after checkout
    KUBECONFIG_PATH = '/var/jenkins_home/kube/config'   // rewritten for k3d inside the pipeline
    K8S_NAMESPACE   = 'default'
    K3D_CLUSTER     = 'devsecops-cluster'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
        script {
          def shortSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.IMAGE = "${APP_NAME}:${shortSha}"
          echo "Building ${env.IMAGE}"
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh """
          docker build -t ${IMAGE} .
          docker image ls | head -n 5
        """
      }
    }

    stage('Trivy Scan (blocking)') {
      steps {
        // We only want failure if Trivy finds HIGH/CRITICAL; no custom summary text
        sh """
          trivy image --severity HIGH,CRITICAL --exit-code 1 --no-progress ${IMAGE}
        """
      }
    }

    stage('Import Image to k3d') {
      steps {
        sh """
          k3d version
          k3d image import ${IMAGE} -c ${K3D_CLUSTER}
        """
      }
    }

    stage('Fix kubeconfig for Jenkins container') {
      steps {
        sh '''
          set -eux
          mkdir -p /var/jenkins_home/kube

          # Copy host kubeconfig from the mount to a path we control
          if [ -f /var/jenkins_home/.kube/config ]; then
            cp /var/jenkins_home/.kube/config ${KUBECONFIG_PATH}
          else
            echo "ERROR: /var/jenkins_home/.kube/config not found (mount $HOME/.kube)."
            exit 1
          fi

          # Rewrite API endpoint to the k3d serverlb (matches cluster cert SANs)
          sed -E "s#https://[^:]+:[0-9]+#https://k3d-${K3D_CLUSTER}-serverlb:6443#g" -i ${KUBECONFIG_PATH}

          echo "Using KUBECONFIG=${KUBECONFIG_PATH}"
          KUBECONFIG=${KUBECONFIG_PATH} kubectl cluster-info
        '''
      }
    }

    stage('Helm Deploy') {
      steps {
        sh """
          helm version
          kubectl version --client

          set -e
          kubectl get ns ${K8S_NAMESPACE} || kubectl create ns ${K8S_NAMESPACE}

          helm upgrade --install ${APP_NAME} ./task-tracker-chart -n ${K8S_NAMESPACE}
          kubectl rollout status deploy/${APP_NAME} -n ${K8S_NAMESPACE} --timeout=120s
        """
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          set -e
          for i in $(seq 1 15); do
            if curl -fsS http://localhost:8080/health >/dev/null; then
              echo "Smoke OK: http://localhost:8080/health"
              exit 0
            fi
            echo "Waiting for app... ($i/15)"; sleep 4
          done
          echo "Smoke test failed"
          exit 1
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Deployed ${IMAGE} to ${K3D_CLUSTER} (${K8S_NAMESPACE})"
    }
    failure {
      echo "❌ Build failed — see the stage logs above."
    }
  }
}

