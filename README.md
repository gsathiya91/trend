# Trend App

## Overview

This project demonstrates a complete DevOps lifecycle for a web application using modern cloud‑native tooling. The goal is to automatically build, containerize, publish, and deploy the application to a Kubernetes cluster on AWS whenever code is pushed to GitHub.

## Automation Flow

GitHub Push → Jenkins CI Pipeline → Docker Build → DockerHub Push → AWS EKS Deploy → Public LoadBalancer URL

---

## 1) Clone Repository

```bash
git clone https://github.com/gsathiya91/trend.git
cd trend
```
---

## 2) Dockerize the Application

Build image:

```bash
docker build -t trend-app .
```

Run locally:

```bash
docker run -d -p 3000:3000 trend-app
```

Open → [http://localhost:3000](http://localhost:3000)

---

## 3) Push Image to DockerHub

Login:

```bash
docker login
```

Tag image:

```bash
docker tag trend-app <dockerhubusername>/trend-app:latest
```

Push image:

```bash
docker push <dockerhubusername>/trend-app:latest
```

---

## 4) Create Infrastructure using Terraform

Terraform creates VPC, subnet, internet gateway, security group and EC2 (Jenkins server).

```bash
cd terraform
terraform init
terraform apply
```

After completion copy the **EC2 Public IP**.

---

## 5) Access Jenkins Server

Open browser:

```
http://<EC2-PUBLIC-IP>:8080
```

Get password (SSH into EC2):

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install **Suggested Plugins**.

---

## 6) Install Required Tools in Jenkins EC2

SSH into server:

```bash
ssh -i <key>.pem ubuntu@<EC2-PUBLIC-IP>
```

### Install Docker

```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo chmod 666 /var/run/docker.sock
sudo systemctl restart docker
sudo systemctl restart jenkins
```

### Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
```

### Install kubectl

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

---

## 7) Add Credentials in Jenkins

Go to:
**Manage Jenkins → Credentials → Global → Add Credentials**

### A) GitHub Credential

Type: Username + Password (GitHub username + Personal Access Token)

### B) DockerHub Credential

* Username: `sathiyagph`
* Password: DockerHub **Access Token**
* ID: `dockerhub`

### C) AWS Credentials

Add your IAM Access Key and Secret Key.

---

## 8) Create EKS Cluster

From your local machine or EC2:

```bash
eksctl create cluster --name trend-cluster --region ap-south-1
```

Configure access on Jenkins server:

```bash
sudo su - jenkins
aws configure
aws eks --region ap-south-1 update-kubeconfig --name trend-cluster
kubectl get nodes
```

(You should see nodes in Ready state)

---

## 9) Kubernetes Deployment Files

### deployment.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trend-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: trend
  template:
    metadata:
      labels:
        app: trend
    spec:
      containers:
      - name: trend
        image: dockerhubusername/trend-app:latest
        ports:
        - containerPort: 3000
```

### service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: trend-service
spec:
  type: LoadBalancer
  selector:
    app: trend
  ports:
  - port: 80
    targetPort: 3000
```

---

## 10) Create Jenkins Pipeline Job

Jenkins → **New Item → Pipeline → Enter Name**

Configure:

* Pipeline script from SCM
* SCM: Git
* Repo URL: github repo url
* Branch: main

Jenkins will automatically read `Jenkinsfile` from repo.

---

## 11) Add GitHub Webhook (Auto Build)

GitHub → Repo → Settings → Webhooks → Add webhook

Payload URL:

```
http://<EC2-PUBLIC-IP>:8080/github-webhook/
```

Content type: `application/json`
Event: **Just push event**

In Jenkins job enable:
**GitHub hook trigger for GITScm polling**

Also open port 8080 in EC2 Security Group.

---

## 12) Run Deployment Automatically

Push any change:

```bash
git add .
git commit -m "deploy"
git push
```

Jenkins will automatically:

1. Pull code
2. Build Docker image
3. Push to DockerHub
4. Deploy to Kubernetes

---

## 13) Get Public Application URL

On Jenkins server:

```bash
kubectl get svc
```

Wait 2–5 minutes and copy **EXTERNAL-IP**.

Open:

```
http://<EXTERNAL-IP>
```

Your application is now live on AWS Kubernetes 🎉

---

## Final CI/CD Flow

GitHub Push → Jenkins Build → Docker Image → DockerHub → Kubernetes EKS → LoadBalancer URL
