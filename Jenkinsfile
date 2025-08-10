pipeline {
  agent any
  environment {
    IMAGE_NAME = "task-tracker:build-${BUILD_NUMBER}"
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }    // <-- key change
    }
    stage('Build Docker Image') {
      steps { sh 'docker build -t $IMAGE_NAME .' }
    }
    stage('Scan with Trivy (non-blocking)') {
      steps { sh 'trivy image $IMAGE_NAME || true' }
    }
    stage('Import Image to k3d') {
      steps { sh 'k3d image import $IMAGE_NAME -c devsecops-cluster' }
    }
    stage('Deploy with Helm') {
      steps { sh 'helm upgrade --install task-tracker ./task-tracker-chart' }
    }
  }
}