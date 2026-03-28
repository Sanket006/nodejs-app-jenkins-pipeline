# ⚙️ Student App — Jenkins CI/CD Pipeline to AWS EKS

<div align="center">

![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![AWS EKS](https://img.shields.io/badge/AWS_EKS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)

*Jenkins declarative pipeline — builds Docker images for a three-tier app (Spring Boot + React + MariaDB), pushes to DockerHub, and deploys to AWS EKS*

</div>

---

## 📌 Overview

This repository contains a **Jenkins declarative CI/CD pipeline** (`simple_deploy_jenkinsfile.jdp`) that automates the full delivery workflow for a three-tier Student Registration application — from pulling source code to deploying containerized services on an **AWS EKS** cluster.

| Stage | What Happens |
|---|---|
| Pull Code | Clone `main` branch from GitHub |
| Build Backend Image | `docker build` from `backend/` → `sanket006/easy-backend:latest` |
| Push Backend Image | Push to DockerHub, remove local image |
| Build Frontend Image | `docker build` from `frontend/` → `sanket006/easy-frontend:latest` |
| Push Frontend Image | Push to DockerHub, remove local image |
| Configure Kubeconfig | `aws eks update-kubeconfig` for `demo-eks-cluster` in `ap-south-1` |
| Deploy to Kubernetes | `kubectl apply -f simple-deploy/` — applies all 3 pod + service manifests |

---

## 🏗️ Architecture

```
Developer pushes to GitHub (main branch)
              │
              ▼
     ┌─────────────────┐
     │   Jenkins Server │
     └────────┬─────────┘
              │
     ┌────────▼─────────────────────────────────────┐
     │  Pipeline Stages                              │
     │                                               │
     │  1. Pull Code ──────── git clone (main)       │
     │  2. Build Backend ──── docker build backend/  │
     │  3. Push Backend ───── docker push → DockerHub│
     │  4. Build Frontend ─── docker build frontend/ │
     │  5. Push Frontend ──── docker push → DockerHub│
     │  6. Kubeconfig ──────── aws eks update-kubeconfig│
     │  7. K8s Deploy ──────── kubectl apply -f      │
     └────────────────────────────┬──────────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │   AWS EKS Cluster          │
                    │   demo-eks-cluster         │
                    │   Region: ap-south-1       │
                    │                            │
                    │  frontend-pod (Port 80)    │ ← LoadBalancer
                    │  backend-pod  (Port 8080)  │ ← NodePort
                    │  db-pod       (Port 3306)  │ ← ClusterIP
                    └────────────────────────────┘
```

---

## 📁 Repository Structure

```
nodejs-app-jenkins-pipeline/
│
├── simple_deploy_jenkinsfile.jdp      # Jenkins declarative pipeline
│
├── simple-deploy/                     # Kubernetes manifests (Pod + Service)
│   ├── backend-pod.yaml               # Spring Boot pod + NodePort service
│   ├── frontend-pod.yaml              # React frontend pod + LoadBalancer service
│   └── db-pod.yaml                    # MariaDB pod + ClusterIP service
│
├── backend/                           # Spring Boot application + Dockerfile
├── frontend/                          # React application + Dockerfile
│
├── README.simple_deploy_jenkinsfile.md  # Pipeline documentation
└── README.md
```

---

## 🔄 Jenkinsfile (`simple_deploy_jenkinsfile.jdp`)

```groovy
pipeline {
    agent any

    stages {
        stage('Pull Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Sanket006/nodejs-app-jenkins-pipeline.git'
            }
        }

        stage('Build Backend Image') {
            steps {
                sh '''cd backend
                docker build -t sanket006/easy-backend:latest .'''
            }
        }

        stage('Push Backend Image') {
            steps {
                sh '''docker push sanket006/easy-backend:latest
                docker rmi -f sanket006/easy-backend:latest'''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''cd frontend
                docker build -t sanket006/easy-frontend:latest .'''
            }
        }

        stage('Push Frontend Image') {
            steps {
                sh '''docker push sanket006/easy-frontend:latest
                docker rmi -f sanket006/easy-frontend:latest'''
            }
        }

        stage('Configure Kubeconfig') {
            steps {
                sh '''aws eks update-kubeconfig --region ap-south-1 --name demo-eks-cluster
                kubectl get nodes'''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f simple-deploy/'
            }
        }
    }
}
```

---

## ☸️ Kubernetes Manifests (`simple-deploy/`)

### `backend-pod.yaml` — Spring Boot backend

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  labels:
    app: backend-app
spec:
  containers:
    - name: backend-app
      image: sanket006/easy-backend:latest
      ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend-app
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
```

### `frontend-pod.yaml` — React frontend

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-app
  labels:
    app: frontend-app
spec:
  containers:
    - name: frontend-app
      image: sanket006/easy-frontend:latest
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend-app
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
```

### `db-pod.yaml` — MariaDB database

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  labels:
    app: easy-crud-db
spec:
  containers:
    - name: mysql-container
      image: mariadb:latest
      ports:
        - containerPort: 3306
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: "<use-a-k8s-secret-here>"   # ⚠️ see security notes below
        - name: MYSQL_DATABASE
          value: student_db
---
apiVersion: v1
kind: Service
metadata:
  name: db-svc
spec:
  selector:
    app: easy-crud-db
  ports:
    - port: 3306
      targetPort: 3306
```

---

## 🚀 Getting Started

### Prerequisites

- Jenkins server (v2.x+) with Pipeline plugin
- Docker installed on the Jenkins agent
- AWS CLI configured on the Jenkins agent with EKS access
- `kubectl` installed on the Jenkins agent
- DockerHub account — credentials stored in Jenkins Credentials Store

### Setup Steps

**1. Clone the repo**
```bash
git clone https://github.com/Sanket006/nodejs-app-jenkins-pipeline.git
```

**2. Store credentials in Jenkins**

Go to **Jenkins → Manage Jenkins → Credentials → Add**:
- DockerHub: Kind `Username with password`, ID `dockerhub-creds`
- AWS: Kind `AWS Credentials`, ID `aws-creds`

**3. Create a Pipeline job**

- New Item → Pipeline
- Pipeline definition: *Pipeline script from SCM*
- SCM: Git → repo URL → branch `main`
- Script Path: `simple_deploy_jenkinsfile.jdp`

**4. Trigger the pipeline**

Push a commit to `main` — or click **Build Now** to run manually.

**5. Access the deployed application**

```bash
# Get the frontend LoadBalancer external IP
kubectl get svc frontend-svc

# Access at:
http://<EXTERNAL-IP>:80
```

---

## ⚠️ Known Issues & Improvements

Based on the current pipeline, here are recommended improvements for production use:

| Issue | Location | Fix |
|---|---|---|
| `latest` tag used for all images | `Jenkinsfile` | Use `${BUILD_NUMBER}` or Git commit SHA for traceable, rollback-safe tags |
| Hardcoded DockerHub credentials | `Jenkinsfile` | Wrap `docker push` in `withCredentials([usernamePassword(...)])` block |
| Hardcoded DB password in pod YAML | `simple-deploy/db-pod.yaml` | Use a Kubernetes `Secret` with `secretKeyRef` instead of plain `value` |
| Pods used instead of Deployments | `simple-deploy/` | Replace `Pod` with `Deployment` for self-healing and rolling update support |
| No tests or image scanning before deploy | `Jenkinsfile` | Add unit test stage and Trivy image scan stage before push |
| Hardcoded EKS cluster name and region | `Jenkinsfile` | Parameterize with `parameters { string(...) }` or Jenkins environment variables |

---

## 🔗 Related Projects

| Repo | Description |
|---|---|
| [student-app-kubernetes](https://github.com/Sanket006/student-app-kubernetes) | Full Kubernetes manifests (Deployments, HPA, Secrets, PVC) for this app |
| [student-app-docker-compose](https://github.com/Sanket006/student-app-docker-compose) | Same app running locally via Docker Compose |
| [jenkins-cicd-pipelines](https://github.com/Sanket006/jenkins-cicd-pipelines) | Full Jenkins pipeline collection including the Terraform EKS pipeline |
| [terraform-aws-iac](https://github.com/Sanket006/terraform-aws-iac) | Terraform to provision the EKS cluster this pipeline deploys to |

---

## 👨‍💻 Author

**Sanket Ajay Chopade** — DevOps Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/sanketchopade07)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/Sanket006)
