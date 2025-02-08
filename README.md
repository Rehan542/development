AWS EKS Web Application Deployment
This repository contains Infrastructure as Code (IaC) and configuration files for deploying a static web application on Amazon EKS (Elastic Kubernetes Service) with monitoring capabilities.
Table of Contents

Prerequisites
Components
Infrastructure Setup
Application Deployment
Monitoring Stack
Directory Structure

Prerequisites

AWS CLI configured with appropriate credentials
Terraform (latest version)
kubectl
Docker
Access to a Docker registry (Docker Hub or similar)

Components
1. Terraform Scripts
Provisions an AWS EKS cluster with necessary networking components:

VPC
Subnets
IAM roles
EKS cluster

2. Kubernetes Manifests

Deployment configuration
Service configuration with LoadBalancer
(Optional) Ingress configuration

3. Monitoring Stack

Prometheus configuration
Grafana setup (for metrics visualization)

4. Dockerfile
Container configuration for the static web application
Infrastructure Setup
Setting up EKS Cluster

Navigate to the terraform directory:

bashCopycd terraform

Initialize Terraform:

bashCopyterraform init

Apply the configuration:

bashCopyterraform apply -auto-approve
Main Terraform Configuration (main.tf)
hclCopyprovider "aws" {
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
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = element(["us-east-1a", "us-east-1b"], count.index)
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
Application Deployment
1. Build and Push Docker Image
bashCopydocker build -t YOUR_DOCKER_HUB/web-app:latest .
docker push YOUR_DOCKER_HUB/web-app:latest
2. Deploy to Kubernetes
bashCopy# Configure kubectl
aws eks --region us-east-1 update-kubeconfig --name web-app-cluster

# Apply deployments
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
Kubernetes Deployment Configuration (deployment.yaml)
yamlCopyapiVersion: apps/v1
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
Service Configuration (service.yaml)
yamlCopyapiVersion: v1
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
Monitoring Stack
Setting up Prometheus
bashCopy# Create monitoring namespace
kubectl create namespace monitoring

# Apply Prometheus configuration
kubectl apply -f monitoring/prometheus-config.yaml
Prometheus Configuration (prometheus-config.yaml)
yamlCopyapiVersion: v1
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
Directory Structure
Copy.
├── README.md
├── terraform/
│   └── main.tf
├── kubernetes/
│   ├── deployment.yaml
│   └── service.yaml
├── monitoring/
│   └── prometheus-config.yaml
└── Dockerfile
License
This project is licensed under the MIT License - see the LICENSE file for details.
