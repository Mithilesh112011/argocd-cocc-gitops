# COCC GitOps CD Pipeline (Argo CD + Kustomize)

This repo contains the GitOps configuration to deploy two containers from the GitLab
container registry using **Argo CD** for continuous delivery and **Kustomize** for
manifest management.

## Structure
```
apps/
  container-app/
    base/                 # environment-agnostic manifests
      deployment.yaml
      service.yaml
      kustomization.yaml
    overlays/
      dev/                # dev-specific patches (namespace, image tags)
        kustomization.yaml
argocd/
  application.yaml         # Argo CD Application resource
```

## Before you deploy
1. Replace all `<group>/<project>/<image-1|2>` placeholders in
   `apps/container-app/base/deployment.yaml` and
   `apps/container-app/overlays/dev/kustomization.yaml` with your actual
   GitLab registry image paths (e.g. `registry.gitlab.com/cocc/myapp/service1`).
2. Replace `<this-repo-name>` in `argocd/application.yaml` with the actual repo name
   once pushed to GitHub.
3. Create the image pull secret for GitLab's registry in the target namespace:
   ```bash
   kubectl create secret docker-registry gitlab-registry-secret \
     --docker-server=registry.gitlab.com \
     --docker-username=<gitlab-user-or-deploy-token> \
     --docker-password=<gitlab-token> \
     --docker-email=<your-email> \
     -n container-app-dev
   ```
   Consider using a secrets controller (External Secrets Operator, Sealed Secrets,
   or Vault) instead of committing/creating secrets manually for production use.

## Deploy
```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f argocd/application.yaml -n argocd
argocd app sync container-app-dev
argocd app get container-app-dev
```

## Validate
```bash
kubectl get pods -n container-app-dev
kubectl get applications -n argocd
```

With `syncPolicy.automated` (prune + selfHeal) enabled, Argo CD will continuously
reconcile the cluster state to match this Git repo — any manual drift is
auto-corrected, and new commits (or image tag bumps) trigger automatic redeployment.
