AWS EKS Web Application Deployment
This repository contains Infrastructure as Code (IaC) and configuration files for deploying a static web application on Amazon EKS with monitoring capabilities.
Components

Terraform Scripts: AWS EKS cluster provisioning
Kubernetes Manifests: Deployment and Service configurations
Monitoring Stack: Prometheus and Grafana setup
Dockerfile: Web application containerization

Prerequisites

AWS CLI configured with appropriate credentials
Terraform installed
kubectl installed
Docker installed and configured
Access to a Docker registry (e.g., Docker Hub)

1. Infrastructure Provisioning with Terraform
Create main.tf:
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
  vpc_id           = aws_vpc.main.id
  cidr_block       = "10.0.${count.index}.0/24"
  availability_zone = element(["us-east-1a", "us-east-1b"], count.index)
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
Initialize and apply Terraform:
bashCopyterraform init
terraform apply -auto-approve
2. Kubernetes Configuration
Deployment Configuration
Create deployment.yaml:
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
Service Configuration
Create service.yaml:
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
Apply the Kubernetes configurations:
bashCopykubectl apply -f deployment.yaml
kubectl apply -f service.yaml
3. Monitoring Setup
Create prometheus-config.yaml:
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
Deploy monitoring:
bashCopykubectl create namespace monitoring
kubectl apply -f prometheus-config.yaml
4. Application Containerization
Create Dockerfile:
dockerfileCopyFROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
Build and push the Docker image:
bashCopydocker build -t YOUR_DOCKER_HUB/web-app:latest .
docker push YOUR_DOCKER_HUB/web-app:latest
5. Deployment Steps

Provision the EKS cluster:
bashCopyterraform apply

Configure kubectl:
bashCopyaws eks --region us-east-1 update-kubeconfig --name web-app-cluster

Deploy the application:
bashCopykubectl apply -f deployment.yaml

Setup monitoring:
bashCopykubectl apply -f prometheus-config.yaml


Verification
To verify the deployment:

Check the EKS cluster status:
bashCopykubectl get nodes

Verify pod status:
bashCopykubectl get pods

Check service status:
bashCopykubectl get services


Cleanup
To clean up resources:
bashCopykubectl delete -f deployment.yaml
kubectl delete -f service.yaml
terraform destroy -auto-approve
