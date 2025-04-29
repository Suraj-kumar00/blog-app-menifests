All the menifests files for deploying the blog application on kubernetes uisng ArgoCD(GitOps Tool) and tekton CI/CD pipeline

# Architecture Diagram

![architecture-diagram](./Assets/blog-app-gitops-architecture.png)

---

I can see from your screenshot that you have the manifests repository structure with directories for argocd, docker, kubernetes, and tekton. Based on your progress and what's remaining, here's a step-by-step implementation plan:

1. **Kubernetes Setup**

   - Ensure your local Kubernetes cluster (Minikube/K3d) is running
   - Apply the Kubernetes manifests from your private repo (postgres-statefulset.yaml, rails-deployment.yaml, etc.)

   ```
   kubectl apply -f kubernetes/namespace.yaml
   kubectl apply -f kubernetes/postgres-statefulset.yaml
   kubectl apply -f kubernetes/rails-deployment.yaml
   kubectl apply -f kubernetes/ingress.yaml
   ```

2. **Deploy ArgoCD**

   - Install ArgoCD in your cluster

   ```
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

   - Apply your ArgoCD config files

   ```
   kubectl apply -f argocd/argocd-cm.yaml
   kubectl apply -f argocd/argocd-rbac-cm.yaml
   kubectl apply -f argocd/repository.yaml
   ```

   - Create the application in ArgoCD

   ```
   kubectl apply -f argocd/application.yaml
   ```

3. **Deploy Tekton**

   - Install Tekton Pipelines

   ```
   kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
   ```

   - Install Tekton Dashboard

   ```
   kubectl apply -f https://storage.googleapis.com/tekton-releases/dashboard/latest/tekton-dashboard-release.yaml
   ```

   - Apply your Tekton configs

   ```
   kubectl apply -f tekton/pipeline.yaml
   kubectl apply -f tekton/pipelinerun.yaml
   ```

4. **Setup GitHub Webhook (optional)**

   - Configure GitHub webhook to trigger Tekton pipeline when changes are pushed to your public application repository

5. **Configure Auto-Sync in ArgoCD**

   - This is likely already defined in your argocd/application.yaml with `syncPolicy.automated: true`
   - When set, ArgoCD will automatically detect changes in your Git repository and apply them

6. **Complete CI/CD Flow**
   - When code is pushed to the public repo, Tekton pipeline is triggered
   - Pipeline clones the repo, builds a new Docker image, and pushes to Docker Hub with a new tag
   - Pipeline updates the image tag in the Kubernetes manifests in the private repo
   - ArgoCD detects the change in manifests and automatically deploys the new version

The pipeline.yaml you have open defines the workspaces for shared storage, Docker credentials, and SSH keys needed to access your repositories. The actual pipeline steps would include:

1. Clone the application code
2. Build the Docker image
3. Push to Docker Hub
4. Update deployment manifests with the new image tag
5. Push changes to the private repository

Would you like me to examine any specific file to provide more detailed guidance on how to set it up? I'd need access to your tekton/pipeline.yaml and tekton/pipelinerun.yaml files to give more specific instructions.
