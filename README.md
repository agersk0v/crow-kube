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

The traffic path is: public IP → router → Windows port 8080 → Docker Desktop LoadBalancer → Traefik → Ingress → frontend → API. PostgreSQL, the API, and the Kubernetes API are not forwarded publicly.

After the app deployments are ready:

```powershell
kubectl apply -f traefik.yaml
kubectl rollout status deployment/traefik -n traefik --timeout=180s
kubectl apply -f ingress.yaml
kubectl get service traefik -n traefik
```

Before applying the updated Service, stop any old Traefik port-forward command with Ctrl+C so port 8080 is free. The Service uses type LoadBalancer with port 8080; Docker Desktop provides the host listener. Open http://localhost:8080. No forwarding terminal is needed. The hostless Ingress accepts your IP directly; no hosts-file entry or domain is required. It routes every path, including /api, to the frontend, which forwards API requests internally. Traefik uses the standard Kubernetes Ingress provider; no Helm or custom resource definitions are required, and its dashboard is not enabled.

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

Windows, Docker Desktop, and Kubernetes must stay running. No kubectl port-forward process is required, including after Traefik pod replacements. Public-IP changes also change the URL.

If the Service stays Pending or localhost:8080 does not respond, inspect Docker Desktop's Kubernetes state and ensure another process is not using port 8080. If localhost works but LAN access does not, check the firewall and Docker Desktop's host binding: a localhost-only listener cannot accept the router's forwarded traffic. This depends on the Docker Desktop backend/version; a LoadBalancer address of localhost alone does not prove LAN access. Verify from another LAN device and mobile data before assuming public access is restored.

To take the demo offline, remove the router rule. To remove the cluster entry point too, run `kubectl delete service traefik -n traefik`; applying traefik.yaml recreates it. Remove its Windows firewall rule from Administrator PowerShell:

```powershell
Remove-NetFirewallRule -DisplayName "Crow demo HTTP"
```

## Automatic deployment with Flux

Merge the crow-fitness chronological-image-tag PR first. Its next successful main build publishes matching `main-<CI run number>-<full SHA>` tags for frontend and API. The API tag is advertised last, after the frontend matching tag exists. Flux scans the API repository, sorts the extracted run number numerically, and writes that same full image tag into both app manifests in one Git commit. It ignores latest and sha-only tags. With no matching tag yet, image automation reports no match and existing workloads remain unchanged.

The flow is: fitness main → Actions builds both images → chronological tags → Flux commits updated manifests here → Flux applies Git → Kubernetes rolls out pods. Rollouts are not simultaneous; old and new component pods can briefly coexist.

### One-time setup on your Windows cluster

Install the pinned Flux CLI (2.9.6) using the Windows instructions at https://fluxcd.io/flux/installation/. With Docker Desktop Kubernetes running:

```powershell
git pull
kubectl config use-context docker-desktop
flux check --pre
flux install --version=v2.9.6 --components=source-controller,kustomize-controller --components-extra=image-reflector-controller,image-automation-controller
flux check
```

This installs four controllers and their CRDs in flux-system. The Flux installation itself is managed by the CLI; controller upgrades are explicit, not automatic. The application manifests and Flux custom resources stay at this repo's root. No folder restructuring or GitHub Actions runner in your cluster is needed.

Create a fine-grained GitHub token restricted to **crow-kube**, with **Contents: Read and write**. Set an expiration you can maintain. It lets Flux read Git and commit image updates directly to main; branch rules must permit those commits. No token belongs in Git or in chat. Enter the token as the password below:

```powershell
$fluxCredential = Get-Credential -UserName agersk0v -Message "Flux: enter the crow-kube Contents read/write token as password"
flux create secret git crow-kube-git --namespace=flux-system --url=https://github.com/agersk0v/crow-kube --username=$($fluxCredential.UserName) --password=$($fluxCredential.GetNetworkCredential().Password)
Remove-Variable fluxCredential
kubectl apply -f flux.yaml
flux reconcile source git crow-kube
flux reconcile kustomization crow-kube --with-source
flux get all
flux get images all
```

Keep the existing database Secret and PVC. Root kustomization.yaml lists exactly the resources Flux applies; it does not include credentials. Reconciliation uses prune:false so removing a manifest does not automatically delete the database, PVC, namespace, or other managed resources. Intentional resource removal needs a separate manual deletion.

The ImageRepository assumes public GHCR access. If scanning reports unauthorized, make both app packages public. Alternatively create a registry Secret in flux-system and reference it in ImageRepository.spec.secretRef; private pod pulls also need their own pull Secret in crow-fitness as described above.

Flux connects outbound to GitHub/GHCR; no extra router port is needed. Once running, you do not need an open terminal for deployment. Windows and Docker Desktop still need to stay on. A sleeping/offline PC catches up after recovery.

### Inspect, pause, and recover

```powershell
flux get sources git
flux get kustomizations
flux get images all
flux reconcile image repository fitness
flux reconcile image update fitness
```

To pause image updates, run `flux suspend image update fitness` and commit `spec.suspend: true` on ImageUpdateAutomation in flux.yaml so reconciliation retains the pause. To roll back, pause automation first, set both image references to the same older published tag in Git, and push. Otherwise automation selects the highest run number again. To resume, remove that suspend field in Git. Flux corrects manual changes to managed Kubernetes resources to match Git.

Renew an expired Git token by replacing the crow-kube-git Secret securely. If a push is rejected, check token permissions and main branch rules. Pull before editing this repo because Flux may have committed newer image references. Credentials stay in cluster Secrets and must be recreated after a cluster reset.

## Inspect and update

```powershell
kubectl get pods,services,pvc -n crow-fitness
kubectl logs -n crow-fitness deployment/api
kubectl describe pods -n crow-fitness
```

An ImagePullBackOff usually means a missing tag or missing GHCR access. A Pending PVC means no suitable default StorageClass is available; check `kubectl get storageclass`. Docker Desktop normally provides one. The PVC deliberately uses the cluster default rather than hardcoding a provider-specific class.

Before Flux is installed, new GHCR images do not restart running pods automatically. For the initial latest-tag setup, you can pull latest again:

```powershell
kubectl rollout restart deployment/api deployment/frontend -n crow-fitness
kubectl rollout status deployment/api -n crow-fitness --timeout=300s
kubectl rollout status deployment/frontend -n crow-fitness --timeout=180s
```

After Flux is enabled, commit desired manifest changes here instead of applying local changes. Flux updates the imagepolicy-marked image lines automatically to matching chronological tags. Do not remove their setter comments. If using the optional frontend localhost port-forward, reconnect it if its pod is replaced.

PostgreSQL uses one replica, Recreate updates, and a 5 GiB persistent volume mounted at /var/lib/postgresql for PostgreSQL 18. Pod replacements preserve the PVC. This is local storage, not a backup or a highly available database. Deleting the namespace/PVC or resetting Docker Desktop Kubernetes can delete workout data. Public access is opt-in through the forwarding steps above. Flux provides automatic deployment after the one-time setup above. Backups and production hosting setup are not included.
