# DevSecOps Demo — Jenkins + Docker + Trivy + k3d + Helm

A minimal end-to-end pipeline that:
- Builds a Docker image for a Flask app
- Scans the image with **Trivy** (blocks on HIGH/CRITICAL)
- Imports the image into a **k3d** (k3s) cluster
- Deploys with **Helm**
- Smoke-tests the service

> Docker Hub namespace used below: **chenbracha** (change if needed)

---

## Repo Structure

```
.
├─ app/                         # Flask app (requirements.txt, app.py, etc.)
├─ task-tracker-chart/          # Helm chart for deployment
├─ Dockerfile                   # Builds the Flask app container
├─ Dockerfile.jenkins           # Jenkins controller image with CLI tools
├─ Jenkinsfile                  # CI/CD pipeline
└─ README.md                    # This file
```

---

## Prerequisites

- Docker Desktop (with Kubernetes disabled — we use k3d)  
- `docker` CLI available locally
- Optional (local testing): `k3d`, `kubectl`, `helm`  
  > The pipeline runs these from **inside Jenkins**, so local installs are optional.

---

## 1) Build and Push the Jenkins Image

We run Jenkins in a container that already includes: `docker` CLI, `trivy`, `kubectl`, `helm`, `k3d`.

```bash
# from repository root
docker build -t chenbracha/jenkins-devsecops:latest -f Dockerfile.jenkins .
docker push chenbracha/jenkins-devsecops:latest
```

---

## 2) Create a k3d Cluster (once)

Create a 1-server, 2-agent k3d cluster with a LoadBalancer called `k3d-devsecops-cluster-serverlb`.

```bash
k3d cluster create devsecops-cluster \
  --agents 2 \
  --port "8080:30080@loadbalancer" \
  --wait
```

Verify:
```bash
kubectl cluster-info
kubectl get nodes -o wide
```

---

## 3) Run Jenkins (controller in Docker)

Mount the host Docker socket and your kube config so Jenkins can build, scan, and deploy.

```bash
docker network create k3d-devsecops-cluster || true

docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/.kube:/var/jenkins_home/.kube \
  --network k3d-devsecops-cluster \
  chenbracha/jenkins-devsecops:latest
```

Get initial admin password:
```bash
docker exec -it jenkins bash -lc "cat /var/jenkins_home/secrets/initialAdminPassword"
```

Open Jenkins: http://localhost:8080

---

## 4) Configure Jenkins (once)

1. Finish setup wizard → install suggested plugins.  
2. Create a **Pipeline** job → “Pipeline from SCM” → Git: your repo URL.  
3. Save and build.

> The pipeline uses the repo’s **Jenkinsfile** and needs no additional credentials for a public GitHub repo.

---

## 5) Pipeline Overview (Jenkinsfile)

The pipeline stages:

1. **Checkout** – pulls the repo and sets `IMAGE=task-tracker:<short-sha>`
2. **Docker Build** – builds the app image with `Dockerfile`
3. **Trivy Scan (blocking)** – runs:
   ```bash
   trivy image --severity HIGH,CRITICAL --exit-code 1 --no-progress ${IMAGE}
   ```
   - If HIGH/CRITICAL are found → **pipeline fails**
4. **Import Image to k3d** – `k3d image import ${IMAGE} -c devsecops-cluster`
5. **Fix kubeconfig for Jenkins container** – rewrites the API server in kubeconfig to
   `https://k3d-devsecops-cluster-serverlb:6443` (matches cluster cert SANs)
6. **Helm Deploy** – `helm upgrade --install` the chart and wait for rollout
7. **Smoke Test** – hits `http://localhost:8080/health` until OK

> No custom parsing/summaries are required. If Trivy finds HIGH/CRITICAL → it prints its table and Jenkins fails the stage.

---

## 6) App Dockerfile (reference)

Your `Dockerfile` should look like:

```dockerfile
# Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY app/ /app/
RUN pip install --no-cache-dir -r requirements.txt
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

Expose via Kubernetes/Helm on NodePort/LoadBalancer 30080 (already mapped to localhost:8080 by k3d port mapping).

---

## 7) Jenkins Image Dockerfile (reference)

Key tools baked in:

- Docker CLI (client)
- Trivy
- kubectl
- Helm
- k3d

```dockerfile
# Dockerfile.jenkins
FROM jenkins/jenkins:lts
USER root

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl gnupg git jq apt-transport-https lsb-release \
    && rm -rf /var/lib/apt/lists/*

# Docker CLI
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo ${VERSION_CODENAME}) stable" > /etc/apt/sources.list.d/docker.list \
 && apt-get update && apt-get install -y docker-ce-cli \
 && rm -rf /var/lib/apt/lists/*

# Trivy
RUN curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor -o /usr/share/keyrings/trivy.gpg \
 && echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb stable main" > /etc/apt/sources.list.d/trivy.list \
 && apt-get update && apt-get install -y trivy \
 && rm -rf /var/lib/apt/lists/*

# kubectl
ARG KUBECTL_VERSION=v1.31.0
RUN curl -fsSL -o /usr/local/bin/kubectl https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl \
 && chmod +x /usr/local/bin/kubectl

# Helm
ARG HELM_VERSION=v3.14.4
RUN curl -fsSL https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz -o /tmp/helm.tgz \
 && tar -xzf /tmp/helm.tgz -C /tmp \
 && mv /tmp/linux-amd64/helm /usr/local/bin/helm \
 && chmod +x /usr/local/bin/helm \
 && rm -rf /tmp/helm.tgz /tmp/linux-amd64

# k3d
ARG K3D_VERSION=v5.8.3
RUN curl -fsSL https://github.com/k3d-io/k3d/releases/download/${K3D_VERSION}/k3d-linux-amd64 -o /usr/local/bin/k3d \
 && chmod +x /usr/local/bin/k3d

# docker group for jenkins
RUN groupadd -g 999 docker || true && usermod -aG docker jenkins

USER jenkins
```

---

## 8) Troubleshooting

- **Trivy stage fails immediately**  
  That’s expected if HIGH/CRITICAL are found. Fix base image or dependencies, or relax gate:
  ```bash
  trivy image --severity CRITICAL --exit-code 1 ${IMAGE}   # only CRITICAL blocks
  ```
- **`k3d ... no such host`**  
  Ensure the Jenkins container is on the `k3d-devsecops-cluster` network:
  ```bash
  docker network connect k3d-devsecops-cluster jenkins || true
  ```
- **`kubectl cluster-info` cert/SAN errors**  
  The pipeline rewrites kubeconfig to `k3d-<cluster>-serverlb:6443`. Do not change this.

---

## 9) Clean Up

```bash
docker rm -f jenkins || true
docker volume rm jenkins_home || true
k3d cluster delete devsecops-cluster || true
```

---

## 10) CI/CD In Short

- Push to `main` → Jenkins pulls → builds image `task-tracker:<short-sha>`
- Trivy scans, **fails** on HIGH/CRITICAL
- If clean → import into k3d → Helm deploy → smoke test `http://localhost:8080/health`

That’s it. Happy shipping! 🚀
