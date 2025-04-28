# Kubernetes, ArgoCD, and Tekton Deployment Guide

This document provides a comprehensive guide to deploying a Ruby on Rails application with PostgreSQL using Kubernetes, ArgoCD for GitOps, and Tekton for CI/CD pipelines.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Kubernetes Deployment](#kubernetes-deployment)
- [ArgoCD Setup](#argocd-setup)
- [Tekton Pipelines](#tekton-pipelines)
- [End-to-End Workflow](#end-to-end-workflow)
- [Concepts Explained](#concepts-explained)

## Prerequisites

Before getting started, ensure you have the following tools installed:

- Docker
- Kubernetes cluster (Minikube or K3d)
- kubectl
- Git
- GitHub account (for private repository)
- Docker Hub account (for storing container images)

## Kubernetes Deployment

### Step 1: Set Up a Local Kubernetes Cluster

Using Minikube:

```bash
# Start Minikube with adequate resources
minikube start --cpus=4 --memory=8g --disk-size=20g

# Enable the ingress addon
minikube addons enable ingress
```

Using K3d:

```bash
# Create a K3d cluster with port mapping
k3d cluster create blog-cluster -p "80:80@loadbalancer" -p "443:443@loadbalancer" --agents 2
```

### Step 2: Create a Namespace

```bash
# Create the namespace for our application
kubectl apply -f kubernetes/namespace.yaml
```

### Step 3: Deploy PostgreSQL StatefulSet

PostgreSQL is deployed as a StatefulSet to ensure data persistence:

```bash
# Apply the PostgreSQL StatefulSet configuration
kubectl apply -f kubernetes/postgres-statefulset.yaml
```

The StatefulSet ensures:

- Stable network identities for PostgreSQL pods
- Persistent storage with PersistentVolumeClaims
- Ordered deployment and scaling

### Step 4: Deploy Redis

Redis is used for caching and background processing:

```bash
# Apply the Redis deployment
kubectl apply -f kubernetes/redis-deployment.yaml
```

### Step 5: Deploy the Rails Application

```bash
# Apply the Rails deployment configuration
kubectl apply -f kubernetes/rails-deployment.yaml
```

The deployment includes:

- ConfigMaps for environment variables
- Secrets for sensitive data
- Health checks for reliability
- Rolling update strategy for zero-downtime deployments

### Step 6: Configure Ingress

```bash
# Apply the Ingress configuration
kubectl apply -f kubernetes/ingress.yaml
```

The Ingress allows external access to your application through a domain name.

### Step 7: Verify Deployment

```bash
# Check all resources in the namespace
kubectl get all -n blog-app

# Verify pods are running
kubectl get pods -n blog-app

# Check services
kubectl get svc -n blog-app

# Test the Ingress
kubectl get ingress -n blog-app
```

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

### Step 2: Access the ArgoCD Dashboard

```bash
# Port-forward the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
```

Visit http://localhost:8080 and login with username `admin` and the password from the command above.

### Step 3: Configure GitHub Repository

1. Create a private GitHub repository for GitOps (e.g., `blog-app-gitops`)
2. Push your Kubernetes manifests to this repository:
   ```bash
   git clone https://github.com/Suraj-kumar00/blog-app-gitops.git
   mkdir -p blog-app-gitops/kubernetes
   cp -r kubernetes/* blog-app-gitops/kubernetes/
   git add .
   git commit -m "Initial commit of Kubernetes manifests"
   git push
   ```

### Step 4: Configure ArgoCD Repository and Application

Apply the ArgoCD configurations:

```bash
# Apply the repository secret
kubectl apply -f argocd/repository.yaml

# Apply the ArgoCD ConfigMaps
kubectl apply -f argocd/argocd-cm.yaml
kubectl apply -f argocd/argocd-rbac-cm.yaml

# Apply the application configuration
kubectl apply -f argocd/application.yaml
```

### Step 5: Verify ArgoCD Sync

In the ArgoCD UI:

1. Navigate to the "Applications" section
2. Click on the "blog-app" application
3. Check the sync status and resource health
4. If necessary, click "Sync" to manually trigger synchronization

## Tekton Pipelines

Tekton is a Kubernetes-native CI/CD solution that builds and deploys your application.

### Step 1: Install Tekton Pipelines and Dashboard

```bash
# Install Tekton Pipelines
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Install Tekton Dashboard
kubectl apply -f https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Wait for Tekton components to be ready
kubectl wait --for=condition=Ready pods --all -n tekton-pipelines --timeout=300s
```

### Step 2: Install Required Tekton Tasks

Instead of using the default catalog tasks, we'll use our own custom tasks with proper security context settings:

```bash
# Apply the custom git-clone task
kubectl apply -f tekton/git-clone-task.yaml

# Apply the custom kaniko task
kubectl apply -f tekton/kaniko-task.yaml

# Verify the tasks were installed
kubectl get tasks -n tekton-pipelines
```

### Step 3: Set Up SSH Authentication for GitHub

```bash
# Generate an SSH key pair
ssh-keygen -t rsa -b 4096 -C "your-email@example.com" -f github-ssh-key

# Display the public key to add to GitHub
cat github-ssh-key.pub
```

1. Add the public key to your GitHub account:

   - Go to GitHub Settings → SSH and GPG keys → New SSH key
   - Give it a title (e.g., "Tekton Pipeline")
   - Paste the public key content and click "Add SSH key"

2. Create a Kubernetes secret with the SSH key:

```bash
# Apply the SSH key secret YAML
kubectl apply -f tekton/github-ssh-key.yaml
```

### Step 4: Configure Docker Hub Authentication

```bash
# Create a Docker Hub authentication secret
kubectl apply -f tekton/docker-credentials.yaml
```

### Step 5: Create a ServiceAccount

```bash
# Create a ServiceAccount with access to both secrets
kubectl apply -f tekton/pipeline-serviceaccount.yaml
```

### Step 6: Create Pipeline

```bash
# Apply the pipeline definition
kubectl apply -f tekton/pipeline.yaml
```

The pipeline performs the following tasks:

1. Clones your application Git repository using SSH
2. Generates a unique tag for the Docker image
3. Builds and pushes the Docker image to Docker Hub using Kaniko
4. Clones the GitOps repository
5. Updates the manifest with the new image tag
6. Commits and pushes the changes back to the GitOps repository

### Step 7: Run the Pipeline

```bash
# Create a PipelineRun to execute the pipeline
kubectl apply -f tekton/pipelinerun.yaml
```

### Step 8: Monitor the Pipeline

```bash
# List PipelineRuns
kubectl get pipelineruns -n tekton-pipelines

# Get logs from a specific PipelineRun (replace with your PipelineRun name)
tkn pipelinerun logs blog-app-pipeline-run -n tekton-pipelines -f

# Or access the Tekton Dashboard
kubectl port-forward -n tekton-pipelines svc/tekton-dashboard 9097:9097
```

Visit http://localhost:9097 in your browser to see the Tekton Dashboard.

### Handling PodSecurity Policies

In clusters with PodSecurity policies enabled, Tekton pipelines must be configured with proper security contexts. Our setup includes:

1. **Security Context for Tasks**: All task steps have been configured with:

   - `allowPrivilegeEscalation: false`
   - `runAsNonRoot: true`
   - `capabilities.drop: ["ALL"]`

2. **PodTemplate in PipelineRun**: The PipelineRun includes a podTemplate that applies security settings to all pods:

   ```yaml
   podTemplate:
     securityContext:
       runAsNonRoot: true
       runAsUser: 65532
       seccompProfile:
         type: "RuntimeDefault"
     containers:
       - name: step-init
         securityContext:
           allowPrivilegeEscalation: false
           capabilities:
             drop:
               - "ALL"
   ```

3. **Custom Task Definitions**: Instead of using the default Tekton catalog tasks, we've created custom versions with proper security contexts.

### Troubleshooting Common Issues

1. **SSH Authentication failures:**

   - Verify your SSH key is correctly added to GitHub
   - Check if the secret has the correct key format
   - Make sure the key has no passphrase

2. **Docker Hub authentication issues:**

   - Ensure your Docker Hub credentials are correct
   - Consider using a personal access token instead of a password
   - Verify the format of the .dockerconfigjson field

3. **PodSecurity Policy failures:**

   - Check the error message for specific security violations
   - Ensure all containers have proper security contexts set
   - Verify the pipeline is configured to run as non-root user
   - Make sure capabilities are appropriately dropped

4. **Pipeline failures:**

   - Check the logs for the specific task that failed
   - Verify file paths and repository URLs
   - Ensure all workspaces are correctly mounted

5. **Git operations failures:**

   - The SSH key needs to have both read and write access to the repositories
   - Check that the GitHub known_hosts entry is correct

6. **Image build failures:**
   - Verify the Dockerfile exists in the specified path
   - Check for syntax errors in the Dockerfile
   - Ensure the build context is correctly set

After the pipeline succeeds, ArgoCD will detect the changes in the GitOps repository and automatically sync the application to the new version.

## End-to-End Workflow

The complete CI/CD workflow:

1. Developer pushes code to the application repository
2. Tekton pipeline is triggered (manually or via webhooks)
3. Tekton:
   - Clones the application repository
   - Builds a container image using Kaniko
   - Pushes the image to Docker Hub
4. Developer updates the image tag in the GitOps repository
5. ArgoCD detects the change and synchronizes the Kubernetes cluster
6. Kubernetes rolls out the new version of the application

## Concepts Explained

### Kubernetes

Kubernetes is a container orchestration platform that automates the deployment, scaling, and management of containerized applications.

Key components used in our setup:

- **Namespace**: Provides isolation for resources
- **StatefulSet**: Used for PostgreSQL to ensure stable network identities and persistent storage
- **Deployment**: Manages the Rails application replicas
- **Service**: Provides network access to pods
- **ConfigMap**: Stores non-sensitive configuration
- **Secret**: Stores sensitive data
- **Ingress**: Routes external traffic to services

Benefits:

- **Scalability**: Easily scale components independently
- **Reliability**: Self-healing capabilities
- **Portability**: Works across various infrastructure providers

### StatefulSet vs Deployment

- **StatefulSet**: Used for PostgreSQL because it provides:

  - Stable, unique network identifiers
  - Stable, persistent storage
  - Ordered, graceful deployment and scaling
  - Ordered, automated rolling updates

- **Deployment**: Used for the Rails application because it provides:
  - Declarative updates
  - Rolling updates and rollbacks
  - Replica scaling
  - Suitable for stateless applications

### GitOps with ArgoCD

GitOps is an operational framework that uses Git as the single source of truth for declarative infrastructure and applications.

ArgoCD implements GitOps by:

1. Monitoring Git repositories for changes
2. Comparing the desired state (Git) with the actual state (Kubernetes)
3. Automatically syncing the cluster to match the Git repository

Benefits:

- **Versioned**: All changes are tracked in Git
- **Auditable**: Complete history of changes
- **Reversible**: Easy rollbacks to previous states
- **Self-documenting**: Configuration as code

### CI/CD with Tekton

Tekton is a Kubernetes-native framework for creating CI/CD systems.

Key components:

- **Tasks**: Individual steps in a pipeline
- **Pipeline**: A series of tasks
- **PipelineRun**: An execution of a pipeline
- **Workspaces**: Shared storage between tasks

Benefits:

- **Cloud Native**: Runs directly on Kubernetes
- **Scalable**: Leverages Kubernetes for scaling
- **Flexible**: Build custom pipelines with reusable components
- **Extensible**: Use or create tasks from the Tekton Catalog

### Best Practices

1. **Secrets Management**:

   - Never commit secrets to Git
   - Use Kubernetes Secrets or external secret management tools

2. **Resource Management**:

   - Set resource requests and limits for all containers
   - Monitor resource usage

3. **High Availability**:

   - Run multiple replicas of the Rails application
   - Use readiness and liveness probes

4. **Security**:
   - Use least privilege principle for RBAC
   - Regularly update container images
   - Scan images for vulnerabilities
