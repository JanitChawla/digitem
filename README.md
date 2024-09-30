Here’s a structured and clean **README** for your Digitem assignment:

---

# Digitem CI/CD Pipeline and Deployment

## Table of Contents
- [Introduction](#introduction)
- [Jenkins CI/CD Pipeline](#jenkins-cicd-pipeline)
- [NGINX Configuration](#nginx-configuration)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Horizontal Pod Autoscaler (HPA)](#horizontal-pod-autoscaler-hpa)

## Introduction

This project demonstrates the CI/CD pipeline configuration using Jenkins and Docker, NGINX reverse proxy setup with custom error pages and SSL, and a Kubernetes deployment with autoscaling based on CPU usage. 

## Jenkins CI/CD Pipeline

The Jenkins pipeline is configured to build, test, and deploy the Docker image of the application to DockerHub, and then deploy the application on Kubernetes. 

### Pipeline Overview

- **Jenkins URL**: [http://<IP>:8080/login](http://1237:8080/login)  
  **Login**: `<>`  
  **Password**: `<>`

### Pipeline Stages

1. **Clone the Repo**: Pulls the project repository from GitHub.
2. **Dynamic Agent Allocation**: Uses a node labeled `docker-node` to run tests in a dynamic docker agent of node.
3. **Test Stage**: Runs the necessary tests within the cloned repository.
4. **Docker Build and Push**: 
    - A new agent builds the Docker image and pushes it to DockerHub.
    - DockerHub credentials are used for authentication during the image push.

### Jenkins Pipeline Configuration (Declarative Syntax)

```groovy
pipeline {
    agent none
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')
    }

    stages {
        stage('Test') {
            agent {
                node {
                    label 'docker-node'
                }
            }
            steps {
                sh 'git clone https://github.com/JanitChawla/digitem.git'
                sh 'cd digitem/ && pwd && ls && node setup'
            }
        }

        stage('Docker and Kubernetes') {
            agent any
            steps {
                deleteDir()
                sh 'git clone https://github.com/JanitChawla/digitem.git'
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'cd digitem/ && pwd && ls && docker build -t janitchawla/digi:latest .'
                sh 'docker push janitchawla/digi:latest'
            }
        }
    }
}
```

## NGINX Configuration

NGINX is configured as a reverse proxy for the application and serves a custom 404 error page. HTTPS is implemented using self-signed SSL certificates.

### Key Features
- **Reverse Proxy**: Forwards incoming requests to the application.
- **SSL Setup**: Self-signed certificates for HTTPS.
- **Custom 404 Page**: Serves a custom error page when a 404 error occurs.

### NGINX Configuration Example

```nginx
server {
    listen 8443;
    location / {
        proxy_pass http://192.168.49.2:8443;
    }
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/certificate/nginx-certificate.crt;
    ssl_certificate_key /etc/nginx/certificate/nginx.key;

    error_page 404 /error.html;

    location / {
        proxy_pass http://192.168.49.2:31000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location = /error.html {
        root /var/www/html;
        internal;
    }
}
```

## Kubernetes Deployment

The application is deployed on a Kubernetes cluster with the following features:
- **Replicas**: 2 replicas of the application.
- **NodePort Service**: Exposes the application using NodePort.
- **Health Checks**: Readiness probe to check the application status.
- **Autoscaling**: HPA configured to scale based on CPU usage.

### Kubernetes Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: react-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: react-app
  template:
    metadata:
      labels:
        app: react-app
    spec:
      containers:
        - name: react-app
          image: janitchawla/digi:latest
          ports:
            - containerPort: 3000
          imagePullPolicy: Always
          resources:
            limits:
              cpu: 500m
            requests:
              cpu: 200m
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: react-svc
spec:
  ports:
    - name: http
      port: 80
      nodePort: 31000
      protocol: TCP
      targetPort: 3000
  selector:
    app: react-app
  type: NodePort
```

## Horizontal Pod Autoscaler (HPA)

The HPA automatically scales the number of pods based on CPU utilization. It requires the metrics server to be installed in the Kubernetes cluster.

### HPA YAML

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: react-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: react-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

### Additional Notes
- **Prometheus and Grafana**: Attempted to set up for monitoring, but the VM experienced lag. The YAML configuration files are available on the VM for further inspection.