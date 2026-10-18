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

Keep the last command running and open http://localhost:3000. If that port is already occupied, use `3001:3000` and open http://localhost:3001. Ctrl+C stops forwarding, not the app. For internet access through Traefik, use the next section instead.

## Public demo through Ingress and router port forwarding

This exposes the app to everyone, without login: visitors can view and change all demo workouts. Use only disposable data. This initial setup is HTTP, with no domain or TLS; do not use it for personal data or passwords.

The traffic path is: public IP → router → Windows port 8080 → kubectl forwarding → Traefik → Ingress → frontend → API. PostgreSQL, the API, and the Kubernetes API are not forwarded publicly.

After the app deployments are ready:

```powershell
kubectl apply -f traefik.yaml
kubectl rollout status deployment/traefik -n traefik --timeout=180s
kubectl apply -f ingress.yaml
kubectl port-forward -n traefik service/traefik 8080:80 --address 0.0.0.0
```

Keep that terminal running. Open http://localhost:8080. The hostless Ingress accepts your IP directly; no hosts-file entry or domain is required. It routes every path, including /api, to the frontend, which forwards API requests internally. Traefik uses the standard Kubernetes Ingress provider; no Helm or custom resource definitions are required, and its dashboard is not enabled.

On Windows, find your active Ethernet/Wi-Fi IPv4 address with `ipconfig` (for example 192.168.1.50, not a WSL or Docker adapter). Reserve that address in your router's DHCP settings. In a separate Administrator PowerShell terminal, allow inbound HTTP on a trusted Private home-network profile:

```powershell
New-NetFirewallRule -DisplayName "Crow demo HTTP" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8080 -Profile Private
```

Leave Windows Firewall enabled. First test http://YOUR-PC-LAN-IP:8080 from another device on your home network. Then create this TCP forwarding rule in your router:

| Setting | Value |
| --- | --- |
| External/WAN port | 8080 |
| Internal/LAN destination | Your Windows PC's reserved IPv4 address |
| Internal/LAN port | 8080 |
| Protocol | TCP |

Visit http://YOUR-PUBLIC-IP:8080 from a phone using mobile data, with Wi-Fi disabled. Some routers cannot reach their own public IP from inside the home network. If you prefer an address without a port, forward external TCP 80 to internal TCP 8080 instead and use http://YOUR-PUBLIC-IP, if your ISP allows inbound port 80.

If LAN access works but external access does not, compare your router's WAN IPv4 with your public IPv4. Carrier-grade NAT (including WAN addresses in 100.64.0.0/10) or upstream private addresses may prevent direct inbound connections. Double NAT requires forwarding on both routers; carrier-grade NAT usually requires a public IP from your ISP or a tunnel. ISP filtering may also block access. Port forwarding cannot overcome these conditions.

Windows, Docker Desktop, Kubernetes, and the forwarding terminal must stay running. Restart the forwarding command if Traefik's pod is replaced. This is temporary demo access, not an unattended hosting setup. Public-IP changes also change the URL.

To take the demo offline, stop the forwarding command and remove the router rule. Remove its Windows firewall rule from Administrator PowerShell:

```powershell
Remove-NetFirewallRule -DisplayName "Crow demo HTTP"
```

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

PostgreSQL uses one replica, Recreate updates, and a 5 GiB persistent volume mounted at /var/lib/postgresql for PostgreSQL 18. Pod replacements preserve the PVC. This is local storage, not a backup or a highly available database. Deleting the namespace/PVC or resetting Docker Desktop Kubernetes can delete workout data. Public access is opt-in through the forwarding steps above. Automatic deployment, backups, and production hosting setup are not included.
