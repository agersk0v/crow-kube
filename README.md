# Crow Kube

Minimal Docker Desktop Kubernetes setup for Crow Fitness. All manifests live at the repository root. Docker Desktop creates the cluster; these files deploy the app into it.

## Start from PowerShell

Enable Kubernetes in Docker Desktop and wait for it to start. Install Git and kubectl if needed. Both GHCR packages must be published; the manifests pull their latest images.

```powershell
git clone https://github.com/agersk0v/crow-kube.git
cd crow-kube
kubectl config use-context docker-desktop
kubectl get nodes
kubectl apply -f namespace.yaml
```

The node should be Ready. These commands create a new, empty workout database, separate from any existing Docker Compose database.

### Image access

If both packages are public, no registry setup is needed. Otherwise, create a GitHub classic token with read:packages, then run the following. Enter that token as the password in the credential prompt:

```powershell
$ghcrCredential = Get-Credential -UserName agersk0v -Message "GHCR: enter your read:packages token as the password"
kubectl create secret docker-registry ghcr -n crow-fitness --docker-server=ghcr.io --docker-username=$($ghcrCredential.UserName) --docker-password=$($ghcrCredential.GetNetworkCredential().Password)
Remove-Variable ghcrCredential
```

Add the following under the pod's `spec` (alongside `containers`) in both api.yaml and frontend.yaml, using your Secret's name:

```yaml
imagePullSecrets:
  - name: ghcr
```

Do not commit tokens or registry Secrets. Package visibility is controlled in GitHub's package settings, independently of repository visibility.

### Database password (first setup only)

Generate a local password and create the shared Secret. The password is not stored in Git.

```powershell
$databasePassword = [guid]::NewGuid().ToString("N")
kubectl create secret generic database --namespace crow-fitness --from-literal="password=$databasePassword"
Remove-Variable databasePassword
```

Keep this Secret when restarting or updating. Generating a new password does not change PostgreSQL's password in an existing data volume. Back up the Secret securely along with database backups.

### Deploy

```powershell
kubectl apply -f postgres.yaml
kubectl rollout status deployment/postgres -n crow-fitness --timeout=180s
kubectl apply -f api.yaml
kubectl rollout status deployment/api -n crow-fitness --timeout=300s
kubectl apply -f frontend.yaml
kubectl rollout status deployment/frontend -n crow-fitness --timeout=180s
kubectl port-forward -n crow-fitness service/frontend 3000:3000 --address 127.0.0.1
```

Keep the last command running and open http://localhost:3000. If that port is already occupied, use `3001:3000` and open http://localhost:3001. Ctrl+C stops forwarding, not the app. There is no Ingress controller, custom hostname, or HTTPS yet; only the port-forward exposes the app, on localhost.

## Inspect and update

```powershell
kubectl get pods,services,pvc -n crow-fitness
kubectl logs -n crow-fitness deployment/api
kubectl describe pods -n crow-fitness
```

An ImagePullBackOff usually means a missing tag or missing GHCR access. A Pending PVC means no suitable default StorageClass is available; check `kubectl get storageclass`. Docker Desktop normally provides one. The PVC deliberately uses the cluster default rather than hardcoding a provider-specific class.

New GHCR images do not restart running pods automatically. To pull latest again:

```powershell
kubectl rollout restart deployment/api deployment/frontend -n crow-fitness
kubectl rollout status deployment/api -n crow-fitness --timeout=300s
kubectl rollout status deployment/frontend -n crow-fitness --timeout=180s
```

For repeatable deployments, replace latest in both manifests with matching published sha-<full-commit-SHA> tags, then apply the files again. Reconnect port forwarding if its pod is replaced.

PostgreSQL uses one replica, Recreate updates, and a 5 GiB persistent volume mounted at /var/lib/postgresql for PostgreSQL 18. Pod replacements preserve the PVC. This is local storage, not a backup or a highly available database. Deleting the namespace/PVC or resetting Docker Desktop Kubernetes can delete workout data. No public exposure, automatic deployment, backups, or production hosting setup is included.
