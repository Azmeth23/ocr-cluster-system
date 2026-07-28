````markdown
# OCR Microservices Architecture

## Architecture Overview

This project implements an **Optical Character Recognition (OCR)** system using a **microservice architecture** deployed on **Kubernetes** with **Helm**.

The application consists of two independent services:

- **API Gateway** – Built with **FastAPI**, it receives client requests and forwards them to the OCR model.
- **OCR Model Server** – Deployed using **KServe**, it performs OCR inference and returns the extracted text.

Each service is containerized using **Docker**, packaged as a **Helm chart**, and deployed to a **Minikube** Kubernetes cluster. **Prometheus** and **Grafana** provide monitoring and visualization, while **ArgoCD** manages Kubernetes deployments using a GitOps workflow.

---

# Containerization

Both the API Gateway and OCR Model Server are packaged as Docker containers, with each service containing its own:

- `Dockerfile`
- Docker image

The build process is automated using the `build-and-test.sh` script. This script:

- Builds the Docker images
- Creates the required Docker network
- Removes any existing containers
- Starts the OCR Model container
- Starts the API Gateway container
- Performs health checks to verify communication
- Pushes the successfully built images to a private Docker Hub repository

Storing the images in Docker Hub allows Kubernetes to pull the same application images during deployment, ensuring consistency between local development and the cluster.

---

# Infrastructure Setup

The Kubernetes infrastructure is created automatically using the `infrastructure-setup.sh` script.

The script performs the following tasks:

- Starts a local **Minikube** Kubernetes cluster using the Docker driver.
- Adds the required Helm repositories for:
  - ArgoCD
  - kube-prometheus-stack
- Creates the ArgoCD namespace and installs **ArgoCD** for GitOps-based deployment management.
- Creates the monitoring namespace and installs the **kube-prometheus-stack**, which includes:
  - Prometheus
  - Grafana
  - Alertmanager
  - Node Exporter
  - Kubernetes monitoring components

This provides a fully configured Kubernetes environment with deployment management and monitoring capabilities.

---

# Helm Deployment

Each microservice is packaged as an independent Helm chart.

```
helm/
├── api-gateway/
└── ocr-model/
```

The charts are packaged using:

```bash
helm package helm/api-gateway
helm package helm/ocr-model
```

This generates:

```
api-gateway-0.1.0.tgz
ocr-model-0.1.0.tgz
```

The packaged charts are then pushed to the Docker Hub OCI Registry, allowing them to be installed directly into Kubernetes using Helm.

---

# Monitoring

Monitoring is implemented using **Prometheus** and **Grafana**.

The OCR Model Server exposes application metrics through the `/metrics` endpoint. A Kubernetes `ServiceMonitor` resource enables Prometheus to automatically collect these metrics.

The monitoring workflow is:

```
OCR Model
    │
 /metrics
    │
    ▼
ServiceMonitor
    │
    ▼
Prometheus
    │
    ▼
Grafana Dashboard
```

Grafana dashboards provide visibility into:

- OCR inference requests
- Request latency
- CPU utilization
- Memory utilization
- Overall application performance

---
## Containerization Strategy

### a. Base Image Selection

- Both the API Gateway and OCR Model use the lightweight `python:3.12-slim` base image.
- The Python base image was selected because both services are developed in Python and the `slim` variant reduces image size, resulting in faster image downloads and lower storage usage.

### b. Security Considerations

- Only the required application dependencies are installed, reducing unnecessary packages within the container.

### c. Build Optimization Techniques
- Frequently changing application code is copied after dependency installation to improve Docker layer caching and reduce rebuild times.


---
# Testing and Validation

## Docker Validation

The Docker images were built and tested locally using:

```bash
chmod +x build-and-test.sh
./build-and-test.sh
```

Validation included:

- Checking generated Docker images

```bash
docker images
```

- Verifying running containers

```bash
docker ps
```

- Sending OCR requests through Postman to:

```
http://localhost:8000/gateway/ocr
```

Successful responses confirmed that the API Gateway and OCR Model Server communicated correctly over the Docker network.

---

## Infrastructure Validation

The Kubernetes infrastructure was deployed using:

```bash
chmod +x infrastructure-setup.sh
./infrastructure-setup.sh
```

Deployment was verified using:

```bash
minikube status
kubectl get nodes
kubectl get pods -n monitoring
kubectl get pods -n argocd
kubectl get svc -n ocr-app
```

The ArgoCD dashboard was accessed using:

```bash
minikube service argocd-server -n argocd
```

Grafana was accessed using:

```bash
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
```

These checks confirmed that the Kubernetes cluster and supporting infrastructure were operating correctly.

---

## Helm Deployment Validation

The Helm charts were packaged using:

```bash
helm package helm/api-gateway
helm package helm/ocr-model
```

After authenticating with Docker Hub and the OCI registry, the charts were pushed using:

```bash
helm push
```

The applications were then installed directly from Docker Hub using Helm:

```bash
helm install gateway ...
helm install ocr ...
```

Successful installation confirmed that the packaged Helm charts could be retrieved and deployed from the remote OCI registry.

---

## Monitoring Validation

Monitoring resources were verified using:

```bash
kubectl get servicemonitors -A
```

To generate observable metrics, 100 OCR requests were sent to the deployed API Gateway:

```bash
for i in {1..100}; do
    curl -X POST \
      -F "image_file=@Documents/image-with-text.jpg" \
      http://$(minikube ip):30000/gateway/ocr > /dev/null
done
```

The generated requests appeared in Prometheus and Grafana, confirming that application metrics were successfully collected and visualized.
````
