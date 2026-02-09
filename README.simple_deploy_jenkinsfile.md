# README — `simple_deploy_jenkinsfile.jdp`

**Purpose**
- Describe what the pipeline in [simple_deploy_jenkinsfile.jdp](simple_deploy_jenkinsfile.jdp#L1) does and how to run or adapt it.

**What this pipeline does (exact steps)**
- Pulls source from the Git repository (branch `main`).
- Builds a Docker image for the backend from `backend/` and tags it `sanket006/easy-backend:latest`.
- Pushes the backend image to the default Docker registry then removes the local image.
- Builds a Docker image for the frontend from `frontend/` and tags it `sanket006/easy-frontend:latest`.
- Pushes the frontend image to the registry then removes the local image.
- Updates kubeconfig for an AWS EKS cluster (`demo-eks-cluster` in `ap-south-1`).
- Applies Kubernetes manifests from the `simple-deploy/` directory.

**Important hard-coded values to consider changing**
- Docker image names and tag: `sanket006/easy-backend:latest`, `sanket006/easy-frontend:latest` — parameterize these.
- EKS cluster name and region used in `aws eks update-kubeconfig --region ap-south-1 --name demo-eks-cluster`.
- `git` URL and branch (currently `main` and the repo URL defined in the pipeline).
- Using the `latest` tag makes rollbacks and reproducible deploys harder — prefer commit SHA or build number.

**Prerequisites (for Jenkins environment)**
- Jenkins with Pipeline plugin and Git plugin.
- A build agent with Docker available (either Docker installed on agent or use Docker-in-Docker).
- Jenkins credentials for Docker registry (if pushing to a private registry).
- AWS CLI and proper AWS credentials configured on the Jenkins agent (or use Jenkins AWS credentials in pipeline) to run `aws eks update-kubeconfig`.
- `kubectl` installed and available on the agent.

**Recommended pipeline improvements**
- Parameterize registry, image names, and image tags via environment variables or Jenkins parameters.
- Use `withCredentials` and secure Jenkins credentials for Docker and AWS instead of relying on global config.
- Replace `sh` commands that `rmi` images with a cleanup stage that handles errors safely.
- Add image scanning, tests, and health checks before pushing and deploying.
- Use unique tags (commit SHA or `${BUILD_NUMBER}`) instead of `latest`.

**How to run the steps locally (commands mirror the pipeline)**

Build backend image locally:
```bash
cd backend
docker build -t sanket006/easy-backend:latest .
```

Push backend image (login first):
```bash
docker push sanket006/easy-backend:latest
```

Build frontend image locally:
```bash
cd frontend
docker build -t sanket006/easy-frontend:latest .
```

Push frontend image:
```bash
docker push sanket006/easy-frontend:latest
```

Update kubeconfig and list nodes (requires AWS CLI + credentials):
```bash
aws eks update-kubeconfig --region ap-south-1 --name demo-eks-cluster
kubectl get nodes
```

Apply manifests to the cluster:
```bash
kubectl apply -f simple-deploy/
```

**Security & operational notes**
- Do not store AWS or Docker credentials in plaintext in the Jenkinsfile. Use Jenkins credentials store.
- Ensure the Jenkins agent has permission to run Docker and to access ECR/Docker Hub if required.
- Confirm Kubernetes manifests in `simple-deploy/` reference the same registry/image tags used by the pipeline.

**Troubleshooting**
- Docker build fails: verify Dockerfile paths and the build context (`backend/` or `frontend/`).
- Docker push fails: check registry credentials and network connectivity.
- `aws eks update-kubeconfig` fails: check AWS credentials, IAM permissions, and that the EKS cluster exists.
- `kubectl apply` fails: inspect manifest syntax and use `kubectl describe`/`kubectl logs` for debugging pods.

**Next steps (suggested)**
- Parameterize image names and tag.
- Replace hard-coded `aws` and `docker` credentials with secure Jenkins credentials.
- Add test and scan stages prior to pushing images.

**Owner / Maintainer**
- Update this file with contact information for the person or team who owns the Jenkins pipeline.
