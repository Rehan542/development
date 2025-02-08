# Web Application Deployment on AWS EKS with Kubernetes & Monitoring

This repository provides Infrastructure as Code (IaC) and Kubernetes manifests to deploy a web application on AWS EKS, including logging and monitoring setups.

## Features
- **Terraform Scripts**: Provision an EKS cluster on AWS.
- **Kubernetes Manifests**: Deployment, Service, and optional Ingress.
- **Monitoring Stack**: Prometheus and Grafana setup.
- **Dockerfile**: Containerization of the static web page.

---

## 1. Provision AWS EKS with Terraform

### `main.tf`
```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_eks_cluster" "my_cluster" {
  name     = "web-app-cluster"
  role_arn = aws_iam_role.eks_role.arn

  vpc_config {
    subnet_ids = aws_subnet.subnets[*].id
  }
}

resource "aws_iam_role" "eks_role" {
  name = "eksClusterRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_subnet" "subnets" {
  count             = 2
  vpc_id           = aws_vpc.main.id
  cidr_block       = "10.0.${count.index}.0/24"
  availability_zone = element(["us-east-1a", "us-east-1b"], count.index)
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

### Run Terraform Commands:
```sh
terraform init
terraform apply -auto-approve
```

---

## 2. Deploy Web Application with Kubernetes

### `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: YOUR_DOCKER_HUB/web-app:latest
        ports:
        - containerPort: 80
```

### `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### Apply the Kubernetes Manifests:
```sh
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

## 3. Monitoring Stack (Prometheus & Grafana)

### `prometheus-config.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'kubernetes'
        static_configs:
          - targets: ['localhost:9090']
```

### Deploy Monitoring Stack:
```sh
kubectl create namespace monitoring
kubectl apply -f prometheus-config.yaml
```

---

## 4. Containerizing the Web Application

### `Dockerfile`
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

### Build & Push Docker Image:
```sh
docker build -t YOUR_DOCKER_HUB/web-app:latest .
docker push YOUR_DOCKER_HUB/web-app:latest
```

---

## 5. Deployment Steps
1. **Provision EKS using Terraform**:
   ```sh
   terraform apply -auto-approve
   ```
2. **Configure `kubectl` for the EKS cluster**:
   ```sh
   aws eks --region us-east-1 update-kubeconfig --name web-app-cluster
   ```
3. **Deploy Web Application**:
   ```sh
   kubectl apply -f deployment.yaml
   ```
4. **Deploy Monitoring Stack**:
   ```sh
   kubectl apply -f prometheus-config.yaml
   ```

---

## Conclusion
This setup automates the deployment of a web application on AWS EKS with monitoring using Terraform, Kubernetes, Docker, Prometheus, and Grafana. Happy deploying! 🚀
