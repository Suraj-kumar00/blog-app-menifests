# Step-by-Step Tekton Pipeline Setup Guide

This guide provides detailed instructions for setting up Tekton pipelines for a Ruby on Rails application, including troubleshooting common issues.

## Prerequisites

- Kubernetes cluster running (Minikube, K3d, or any other cluster)
- `kubectl` configured to connect to your cluster
- Docker Hub account (for storing container images)
- GitHub account (for accessing the repository)

## Step 1: Install Tekton Pipelines and Dashboard

```bash
# Create a dedicated namespace for Tekton (if not using the default tekton-pipelines)
kubectl create namespace tekton-pipelines

# Install Tekton Pipelines - note the new URL with ghcr.io
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Install Tekton Dashboard
kubectl apply -f https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Wait for Tekton components to be ready
kubectl wait --for=condition=Ready pods --all -n tekton-pipelines --timeout=300s
```

## Step 2: Install Required Tekton Tasks

Tekton requires reusable tasks that must be installed in your cluster:

```bash
# Download and install git-clone task
curl -sL https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml | kubectl apply -f - -n tekton-pipelines

# Download and install kaniko task for building container images
curl -sL https://raw.githubusercontent.com/tektoncd/catalog/main/task/kaniko/0.6/kaniko.yaml | kubectl apply -f - -n tekton-pipelines

# Verify the tasks were installed
kubectl get tasks -n tekton-pipelines
```

## Step 3: Configure Authentication Secrets

### GitHub Authentication Secret

Create a secret for GitHub authentication:

1. Create a GitHub Personal Access Token with `repo` permissions
2. Create a Secret YAML file (github-credentials.yaml):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-credentials
  namespace: tekton-pipelines
  annotations:
    tekton.dev/git-0: https://github.com # This annotation is important
type: kubernetes.io/basic-auth
stringData:
  username: your-github-username
  password: your-github-personal-access-token
```

3. Apply the secret:

```bash
kubectl apply -f tekton/github-credentials.yaml
```

### Docker Hub Authentication Secret

Create a secret for Docker Hub authentication:

1. Create a Secret YAML file (docker-credentials.yaml):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-credentials
  namespace: tekton-pipelines
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "https://index.docker.io/v1/": {
          "username": "your-dockerhub-username",
          "password": "your-dockerhub-password",
          "email": "your-email@example.com"
        }
      }
    }
```

2. Apply the secret:

```bash
kubectl apply -f tekton/docker-credentials.yaml
```

## Step 4: Create a ServiceAccount

Create a ServiceAccount that has access to the GitHub and Docker Hub secrets:

1. Create a ServiceAccount YAML file (pipeline-serviceaccount.yaml):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-account
  namespace: tekton-pipelines
secrets:
  - name: github-credentials
  - name: docker-credentials
```

2. Apply the ServiceAccount:

```bash
kubectl apply -f tekton/pipeline-serviceaccount.yaml
```

## Step 5: Create the Tekton Pipeline

Create a Pipeline that defines the workflow:

1. Create a Pipeline YAML file (pipeline.yaml):

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: blog-app-pipeline
  namespace: tekton-pipelines
spec:
  workspaces:
    - name: shared-workspace
    - name: docker-credentials
  params:
    - name: git-url
      type: string
      description: Git repository URL
      default: https://github.com/Suraj-kumar00/blog-app-gitops.git
    - name: git-revision
      type: string
      description: Git revision to checkout
      default: main
    - name: image-name
      type: string
      description: Name of the image to build
      default: your-dockerhub-username/blog-app
    - name: image-tag
      type: string
      description: Tag for the image
      default: latest
  tasks:
    - name: fetch-repository
      taskRef:
        name: git-clone
        kind: Task # Important: Use Task, not ClusterTask
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)
        - name: depth
          value: "1"
        - name: verbose
          value: "true"

    - name: build-and-push
      taskRef:
        name: kaniko
        kind: Task # Important: Use Task, not ClusterTask
      runAfter:
        - fetch-repository
      workspaces:
        - name: source
          workspace: shared-workspace
        - name: dockerconfig
          workspace: docker-credentials
      params:
        - name: IMAGE
          value: $(params.image-name):$(params.image-tag)
        - name: CONTEXT
          value: $(workspaces.source.path)
        - name: DOCKERFILE
          value: ./Dockerfile
```

2. Apply the Pipeline:

```bash
kubectl apply -f tekton/pipeline.yaml
```

## Step 6: Create a PipelineRun

Create a PipelineRun to execute the pipeline:

1. Create a PipelineRun YAML file (pipelinerun.yaml):

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: blog-app-pipeline-run-
  namespace: tekton-pipelines
spec:
  serviceAccountName: pipeline-account
  pipelineRef:
    name: blog-app-pipeline
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
    - name: docker-credentials
      secret:
        secretName: docker-credentials
  params:
    - name: git-url
      value: https://github.com/Suraj-kumar00/blog-app-gitops.git
    - name: image-name
      value: your-dockerhub-username/blog-app
    - name: image-tag
      value: latest
```

2. Apply the PipelineRun:

```bash
kubectl create -f tekton/pipelinerun.yaml
```

## Step 7: Monitor the Pipeline Execution

### Using kubectl

```bash
# List all PipelineRuns
kubectl get pipelineruns -n tekton-pipelines

# View details of a specific PipelineRun
kubectl describe pipelinerun <pipelinerun-name> -n tekton-pipelines

# View logs of a specific TaskRun
kubectl logs -l tekton.dev/pipelineRun=<pipelinerun-name> -n tekton-pipelines --container step-clone
```

### Using Tekton Dashboard

1. Access the Tekton Dashboard:

```bash
kubectl port-forward -n tekton-pipelines svc/tekton-dashboard 9097:9097
```

2. Open your browser and navigate to http://localhost:9097
3. Navigate to "PipelineRuns" to see all pipeline executions
4. Click on a specific PipelineRun to view its details and logs

## Troubleshooting Common Issues

### Error: ClusterTask not found

**Issue**: `Couldn't retrieve Task "git-clone": clustertasks.tekton.dev "git-clone" not found`

**Solution**:

1. Make sure you've installed the tasks correctly:
   ```bash
   kubectl get tasks -n tekton-pipelines
   ```
2. In your Pipeline YAML, change `kind: ClusterTask` to `kind: Task`

### Error: GitHub Authentication Failure

**Issue**: `fatal: could not read Username for 'https://github.com': No such device or address`

**Solution**:

1. Create a proper GitHub credentials secret with the correct annotation
2. Use a ServiceAccount that references this secret
3. Make sure the PipelineRun uses this ServiceAccount

### Error: Docker Hub Authentication Failure

**Issue**: `unauthorized: incorrect username or password`

**Solution**:

1. Verify your Docker Hub credentials are correct
2. Make sure the Secret is correctly formatted
3. Check that the workspace is mounted correctly in the PipelineRun

### Error: Image Not Found

**Issue**: `failed to pull and unpack image "gcr.io/tekton-releases/..."`

**Solution**:

1. Tekton has moved to GitHub Container Registry (ghcr.io)
2. Make sure you're using the latest installation commands from the official documentation

## Best Practices

1. **Use Task instead of ClusterTask**: Regular Tasks are scoped to a namespace and easier to manage
2. **Never commit secrets to Git**: Keep all your credential files separate
3. **Use ServiceAccounts**: Always run your pipelines with dedicated ServiceAccounts
4. **Parameterize your pipelines**: Make them reusable by defining parameters
5. **Keep resources minimal**: Only request the storage and compute resources you need
