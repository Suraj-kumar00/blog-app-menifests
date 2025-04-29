# Kubernetes, ArgoCD, and Tekton Deployment Guide

This document provides a comprehensive guide to deploying a Ruby on Rails application with PostgreSQL using Kubernetes, ArgoCD for GitOps, and Tekton for CI/CD pipelines.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Environment Variables and Secrets](#environment-variables-and-secrets)
- [ArgoCD Setup](#argocd-setup)
- [Tekton Pipelines](#tekton-pipelines)
- [Security Concerns](#security-concerns)
- [End-to-End Workflow](#end-to-end-workflow)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before getting started, ensure you have the following tools installed:

- Docker
- Kubernetes cluster (Minikube or K3d)
- kubectl
- Git
- GitHub account (for both public and private repositories)
- Docker Hub account (for storing container images)

## Repository Structure

This project uses two separate repositories:

1. **Public Application Repository**: Contains the Ruby on Rails application code

   - URL: `https://github.com/Suraj-kumar00/blog-app-gitops.git`
   - Contents: Ruby on Rails application, Dockerfile, etc.

2. **Private Manifests Repository**: Contains all Kubernetes, ArgoCD, and Tekton configuration files
   - URL: `git@github.com:Suraj-kumar00/blog-app-menifests.git`
   - Contents:
     - `/kubernetes/`: Kubernetes manifests (deployments, services, etc.)
     - `/argocd/`: ArgoCD configuration files
     - `/tekton/`: Tekton pipeline definitions and related files

## Kubernetes Deployment

### Step 1: Set Up a Local Kubernetes Cluster

You can use either Minikube or K3d for local development.

#### Using Minikube:

```bash
# Start Minikube with adequate resources
minikube start --cpus=4 --memory=8g --disk-size=20g

# Enable the ingress addon
minikube addons enable ingress
```

#### Using K3d:

```bash
# Create a K3d cluster with port mapping
k3d cluster create blog-cluster -p "80:80@loadbalancer" -p "443:443@loadbalancer" --agents 2
```

### Step 2: Clone the Private Repository

```bash
# Clone the private repository containing manifests
git clone git@github.com:Suraj-kumar00/blog-app-menifests.git
cd blog-app-menifests
```

### Step 3: Create Namespace

```bash
# Create the namespace for our application
kubectl create namespace blog-app

# Alternatively, if you have a namespace YAML file
kubectl apply -f kubernetes/namespace.yaml
```

This will create a `blog-app` namespace where our application components will be deployed.

### Step 4: Create Secrets and ConfigMaps for Environment Variables

Before deploying the application, you need to create Kubernetes resources to store environment variables. This approach is more secure than using a .env file directly.

```bash
# Create a ConfigMap for non-sensitive environment variables
kubectl create configmap rails-config -n blog-app \
  --from-literal=RAILS_ENV=production \
  --from-literal=POSTGRES_HOST=postgres \
  --from-literal=POSTGRES_DB=blog_production \
  --from-literal=REDIS_URL=redis://redis:6379/1 \
  --from-literal=RAILS_LOG_TO_STDOUT=true \
  --from-literal=RAILS_SERVE_STATIC_FILES=true

# Create a Secret for sensitive environment variables
kubectl create secret generic rails-secrets -n blog-app \
  --from-literal=POSTGRES_USER=dean \
  --from-literal=POSTGRES_PASSWORD=password123 \
  --from-literal=RAILS_MASTER_KEY=8af814b3e08d9e263c6635f231fc541f
```

Alternatively, you can apply these directly from the Kubernetes manifest files:

```bash
# If your rails-deployment.yaml already includes ConfigMap and Secret definitions
kubectl apply -f kubernetes/rails-deployment.yaml -n blog-app
```

> **IMPORTANT**: Make sure sensitive values like passwords and API keys are not committed to Git. Store them securely and create secrets through kubectl commands or a secret management tool.

### Step 5: Deploy PostgreSQL StatefulSet

PostgreSQL is deployed as a StatefulSet to ensure data persistence:

```bash
# Apply the PostgreSQL StatefulSet configuration
kubectl apply -f kubernetes/postgres-statefulset.yaml -n blog-app
```

Verify the deployment:

```bash
kubectl get statefulsets -n blog-app
kubectl get pvc -n blog-app
kubectl get pods -n blog-app -l app=postgres
```

### Step 6: Deploy Redis (if required)

```bash
# Apply the Redis deployment
kubectl apply -f kubernetes/redis-deployment.yaml -n blog-app

# Verify the deployment
kubectl get deployments -n blog-app -l app=redis
kubectl get pods -n blog-app -l app=redis
```

### Step 7: Deploy the Rails Application

```bash
# Apply the Rails deployment configuration
kubectl apply -f kubernetes/rails-deployment.yaml -n blog-app

# Verify the deployment
kubectl get deployments -n blog-app -l app=rails
kubectl get pods -n blog-app -l app=rails
```

### Step 8: Configure Ingress

```bash
# Apply the Ingress configuration
kubectl apply -f kubernetes/ingress.yaml -n blog-app

# Verify the ingress
kubectl get ingress -n blog-app
```

### Step 9: Verify Deployment

```bash
# Check all resources in the namespace
kubectl get all -n blog-app

# Check logs for any errors
kubectl logs -n blog-app deployment/rails-app
```

## Environment Variables and Secrets

### Managing Environment Variables in Kubernetes

Your application needs access to environment variables for configuration. In Kubernetes, these are managed through ConfigMaps (for non-sensitive data) and Secrets (for sensitive data).

The rails-deployment.yaml file includes:

1. A ConfigMap with non-sensitive environment variables:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: rails-config
   data:
     RAILS_ENV: production
     POSTGRES_HOST: postgres
     POSTGRES_DB: blog_production
     # Other non-sensitive variables
   ```

2. A Secret with sensitive information:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: rails-secrets
   type: Opaque
   data:
     POSTGRES_PASSWORD: cG9zdGdyZXM= # Base64 encoded
     RAILS_MASTER_KEY: Y2hhbmdlbWU= # Base64 encoded
   ```

3. Environment variables in the container definition:
   ```yaml
   containers:
     - name: rails
       # ...
       env:
         - name: RAILS_ENV
           valueFrom:
             configMapKeyRef:
               name: rails-config
               key: RAILS_ENV
         - name: POSTGRES_PASSWORD
           valueFrom:
             secretKeyRef:
               name: rails-secrets
               key: POSTGRES_PASSWORD
         # Other environment variables
   ```

### Environment Variables in Docker Build Process

When Tekton builds your Docker image using Kaniko, it doesn't automatically use the .env file. You have two options:

#### Option 1: Add build arguments to your Tekton Pipeline

Update your pipeline.yaml to pass build arguments to the Docker build process:

```yaml
- name: build-and-push
  taskRef:
    name: kaniko
    kind: Task
  params:
    - name: IMAGE
      value: $(params.image-name):$(tasks.generate-tag.results.tag)
    - name: CONTEXT
      value: $(workspaces.source.path)/source
    - name: DOCKERFILE
      value: ./Dockerfile
    - name: EXTRA_ARGS
      value:
        - "--build-arg=RAILS_ENV=production"
        - "--build-arg=RAILS_MASTER_KEY=8af814b3e08d9e263c6635f231fc541f"
```

Make sure your Dockerfile accepts these build arguments:

```Dockerfile
ARG RAILS_ENV
ARG RAILS_MASTER_KEY
ENV RAILS_ENV=${RAILS_ENV}
ENV RAILS_MASTER_KEY=${RAILS_MASTER_KEY}
```

#### Option 2: Create a ConfigMap from your .env file

If you have many environment variables, you can create a ConfigMap from your .env file:

```bash
# Create a ConfigMap from your .env file
kubectl create configmap app-env -n tekton-pipelines --from-file=.env
```

Then modify your pipeline to use this ConfigMap (this requires additional configuration).

### Best Practices for Environment Variables and Secrets

1. **Never commit .env files or secrets to Git**
2. **Use Kubernetes Secrets for sensitive data**
3. **Update secrets and ConfigMaps before deploying the application**
4. **Use build arguments only for necessary build-time variables**
5. **Consider using a secrets management solution** like Vault or Sealed Secrets for production environments

## ArgoCD Setup

ArgoCD implements GitOps practices by continuously synchronizing your Kubernetes cluster with a Git repository.

### Step 1: Install ArgoCD

```bash
# Create the ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD pods to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### Step 2: Create SSH Key for Repository Access

> **IMPORTANT**: This step should be done locally, NEVER commit SSH keys to Git repositories!

```bash
# Generate an SSH key pair
ssh-keygen -t rsa -b 4096 -C "your-email@example.com" -f ~/.ssh/argocd_repo_key

# Display the public key to add to GitHub
cat ~/.ssh/argocd_repo_key.pub
```

Add this public key to your GitHub repository:

1. Go to your private GitHub repository
2. Navigate to Settings → Deploy keys
3. Click "Add deploy key"
4. Paste the public key and give it a title like "ArgoCD Deploy Key"
5. Check "Allow write access" if needed
6. Click "Add key"

### Step 3: Create Repository Secret

Create a Kubernetes secret with your private SSH key to access the repository:

```bash
# Create a Secret with the private key in the argocd namespace
kubectl -n argocd create secret generic blog-repo-ssh \
  --from-file=sshPrivateKey=$HOME/.ssh/argocd_repo_key \
  --from-literal=url=git@github.com:Suraj-kumar00/blog-app-menifests.git
```

**IMPORTANT**: The file `argocd/repository.yaml` in the repository contains a GitHub token. This should NOT be committed to the repository. Instead, use the command above to create the secret.

### Step 4: Configure ArgoCD Repository and Application

Apply the ArgoCD configurations:

```bash
# Apply the repository secret (if you want to use the YAML file instead of the command above)
# DO NOT use this file as-is because it contains credentials!
# kubectl apply -f argocd/repository.yaml -n argocd

# Apply the ArgoCD ConfigMaps (if needed)
kubectl apply -f argocd/argocd-cm.yaml -n argocd
kubectl apply -f argocd/argocd-rbac-cm.yaml -n argocd

# Apply the application configuration
kubectl apply -f argocd/application.yaml -n argocd
```

### Step 5: Access the ArgoCD Dashboard

```bash
# Port-forward the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
```

Visit http://localhost:8080 in your browser and login with:

- Username: `admin`
- Password: The password from the command above

### Step 6: Verify ArgoCD Setup

1. In the ArgoCD UI, navigate to "Applications"
2. You should see the "blog-app" application
3. Click on it to see its status
4. Check if it's synced with your repository

## Tekton Pipelines

Tekton provides a Kubernetes-native CI/CD solution that builds and deploys your application.

### Step 1: Install Tekton Pipelines and Dashboard

```bash
# Install Tekton Pipelines - this creates the tekton-pipelines namespace automatically
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Install Tekton Dashboard
kubectl apply -f https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Wait for Tekton components to be ready
kubectl wait --for=condition=Ready pods --all -n tekton-pipelines --timeout=300s
```

### Step 2: Install Required Tekton Tasks

Tekton uses Tasks as building blocks for pipelines. We need to install the git-clone and kaniko tasks:

```bash
# Install the git-clone task in the tekton-pipelines namespace
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.6/git-clone.yaml -n tekton-pipelines

# Install the kaniko task in the tekton-pipelines namespace
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/kaniko/0.6/kaniko.yaml -n tekton-pipelines

# Verify the tasks
kubectl get tasks -n tekton-pipelines
```

### Step 3: Create SSH Key for GitHub Access

> **IMPORTANT**: This step should be done locally, NEVER commit SSH keys to Git repositories!

```bash
# Generate an SSH key pair for Tekton
ssh-keygen -t rsa -b 4096 -C "your-email@example.com" -f ~/.ssh/tekton_github_key -N ""

# Display the public key to add to GitHub
cat ~/.ssh/tekton_github_key.pub
```

Add this public key to your GitHub accounts:

1. Go to your GitHub account settings
2. Navigate to SSH and GPG keys
3. Click "New SSH key"
4. Paste the public key and give it a title like "Tekton Pipeline Key"
5. Click "Add SSH key"

### Step 4: Create Secrets for Tekton

#### Create GitHub SSH Key Secret

```bash
# Create the SSH key secret in the tekton-pipelines namespace
kubectl create secret generic github-ssh-key -n tekton-pipelines \
  --from-file=id_rsa=~/.ssh/tekton_github_key \
  --from-file=id_rsa.pub=~/.ssh/tekton_github_key.pub \
  --from-literal=known_hosts="github.com ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAq2A7hRGmdnm9tUDbO9IDSwBK6TbQa+PXYPCPy6rbTrTtw7PHkccKrpp0yVhp5HdEIcKr6pLlVDBfOLX9QUsyCOV0wzfjIJNlGEYsdlLJizHhbn2mUjvSAHQqZETYP81eFzLQNnPHt4EVVUh7VfDESU84KezmD5QlWpXLmvU31/yMf+Se8xhHTvKSCZIFImWwoG6mbUoWf9nzpIoaSjB+weqqUUmpaaasXVal72J+UX2B+2RPW3RcT0eOzQgqlJL3RKrTJvdsjE3JEAvGq3lGHSZXy28G3skua2SmVi/w4yCE6gbODqnTWlg7+wC604ydGXA8VJiS5ap43JXiUFFAaQ=="
```

**IMPORTANT**: The file `tekton/github-ssh-key.yaml` in the repository contains a private SSH key. This should NOT be committed to the repository. Instead, use the command above to create the secret.

#### Create Docker Hub Credentials Secret

```bash
# Create a Docker Hub credentials secret in the tekton-pipelines namespace
kubectl create secret docker-registry docker-credentials -n tekton-pipelines \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-dockerhub-username> \
  --docker-password=<your-dockerhub-password> \
  --docker-email=<your-email>
```

**IMPORTANT**: The file `tekton/docker-credentials.yaml` in the repository contains Docker Hub credentials. This should NOT be committed to the repository. Instead, use the command above to create the secret.

### Step 5: Create Service Account for Tekton

```bash
# Apply the pipeline service account in the tekton-pipelines namespace
kubectl apply -f tekton/pipeline-serviceaccount.yaml -n tekton-pipelines
```

#### Create RBAC for Cross-Namespace Access (if needed)

If your Tekton pipeline needs to update resources in other namespaces (like updating the manifests that ArgoCD will sync):

```bash
# Create a role and role binding that allows the pipeline to access argocd resources
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tekton-cross-namespace-access
rules:
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tekton-cross-namespace-access-binding
subjects:
- kind: ServiceAccount
  name: pipeline-account
  namespace: tekton-pipelines
roleRef:
  kind: ClusterRole
  name: tekton-cross-namespace-access
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Step 6: Apply Pipeline Configuration

```bash
# Apply the pipeline definition in the tekton-pipelines namespace
kubectl apply -f tekton/pipeline.yaml -n tekton-pipelines

# If you need to add build arguments for environment variables, edit pipeline.yaml first
# to include the EXTRA_ARGS parameter as shown in the Environment Variables section
```

### Step 7: Create and Run PipelineRun

Update the pipelinerun.yaml file to use the correct repository URLs:

```bash
# Apply the PipelineRun in the tekton-pipelines namespace
kubectl apply -f tekton/pipelinerun.yaml -n tekton-pipelines
```

### Step 8: Access the Tekton Dashboard

```bash
# Port-forward the Tekton Dashboard
kubectl port-forward -n tekton-pipelines svc/tekton-dashboard 9097:9097
```

Visit http://localhost:9097 in your browser to access the Tekton Dashboard.

### Step 9: Verify Pipeline Execution

In the Tekton Dashboard:

1. Navigate to "PipelineRuns"
2. Click on the most recent PipelineRun
3. Check the logs and status of each task

## Security Concerns

### Handling Secrets Securely

The current setup has several security issues:

1. **Secrets in Git Repository**:

   - The repository contains SSH private keys, GitHub tokens, and Docker Hub credentials directly in YAML files.
   - These should NEVER be committed to Git repositories, even private ones.

2. **Proper Approach for Secrets**:
   - Generate keys and credentials locally
   - Create Kubernetes secrets using kubectl commands
   - Reference these secrets in your configurations
   - Use git-crypt or SOPS for encrypting secrets if they must be in Git

### Secure Pipeline Configuration

The pipeline should use least-privilege principles:

1. **Service Account Permissions**: Ensure the service account used by Tekton has only the necessary permissions.

2. **Security Context**: All containers should run with:

   - Non-root user
   - Read-only root filesystem when possible
   - Dropped capabilities
   - No privilege escalation

3. **Workspace Security**: Ensure workspace volumes are properly secured.

## End-to-End Workflow

The complete CI/CD workflow with our setup:

1. Developer pushes code to the public application repository (`blog-app-gitops`)
2. Tekton pipeline is triggered (manually or via webhooks)
3. Tekton:
   - Clones the application repository
   - Builds a Docker image using Kaniko
   - Pushes the image to Docker Hub with a new tag based on commit hash and timestamp
   - Updates the Kubernetes manifests in the private repository with the new image tag
4. ArgoCD detects the changes in the private repository
5. ArgoCD automatically syncs the application based on the updated manifests
6. Kubernetes rolls out the new version of the application

### Auto-Sync with ArgoCD

ArgoCD is configured with automatic sync enabled in the application.yaml:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

This means:

- Changes to the manifests in the Git repository will be automatically applied to the cluster
- Resources that are no longer in Git will be removed from the cluster (prune)
- Manual changes to resources in the cluster will be reverted to match Git (selfHeal)

## Troubleshooting

### ArgoCD Issues

1. **Repository Connection Issues**:

   ```bash
   # Check the ArgoCD repository connection status
   kubectl get secret -n argocd blog-repo -o yaml

   # Check ArgoCD application status
   kubectl get application -n argocd blog-app -o yaml

   # Check logs from ArgoCD repository server
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server
   ```

2. **Application Sync Issues**:

   ```bash
   # Manually sync the application
   argocd app sync blog-app

   # Check the application status
   argocd app get blog-app
   ```

### Tekton Issues

1. **Pipeline Run Failures**:

   ```bash
   # Get the PipelineRun status
   kubectl get pipelinerun -n tekton-pipelines

   # Check the logs of a failed PipelineRun
   tkn pipelinerun logs <pipelinerun-name> -n tekton-pipelines -f
   ```

2. **Task Failures**:

   ```bash
   # Get the TaskRun status
   kubectl get taskrun -n tekton-pipelines

   # Check the logs of a failed TaskRun
   tkn taskrun logs <taskrun-name> -n tekton-pipelines -f
   ```

3. **Workspace Issues**:

   ```bash
   # Check if the PVC for the workspace was created
   kubectl get pvc -n tekton-pipelines
   ```

4. **Secret Access Issues**:
   ```bash
   # Verify the service account has access to the secrets
   kubectl get serviceaccount pipeline-account -n tekton-pipelines -o yaml
   ```

### Kubernetes Issues

1. **Pod Startup Issues**:

   ```bash
   # Check pod status
   kubectl get pods -n blog-app

   # Check pod logs
   kubectl logs <pod-name> -n blog-app

   # Describe the pod for events
   kubectl describe pod <pod-name> -n blog-app
   ```

2. **Service Issues**:

   ```bash
   # Check service endpoints
   kubectl get endpoints -n blog-app
   ```

3. **Ingress Issues**:

   ```bash
   # Check ingress status
   kubectl get ingress -n blog-app

   # Check ingress controller logs
   kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
   ```

Remember to replace placeholders like `<your-dockerhub-username>`, `<your-dockerhub-password>`, and `<your-email>` with your actual information.

**IMPORTANT**: Always handle secrets securely. The current repository has secrets committed directly in YAML files, which is a significant security risk.
