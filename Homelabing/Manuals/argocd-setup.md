# Argo CD on the k3s Cluster

Installed on: `k3s-ctrlr` (VM 400), the only node with `kubectl` configured for cluster access.
Reference: [Argo CD official getting started guide](https://argo-cd.readthedocs.io/en/stable/getting_started/)

## 1. Install

```
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

`--server-side --force-conflicts` are required, not optional. Some of Argo CD's CRDs exceed the annotation size limit plain `kubectl apply` enforces, without these flags the install errors out partway through.

Argo CD isn't one process, it's several pods working together inside the `argocd` namespace: `argocd-server` (API/UI), `argocd-repo-server` (reads Git), `argocd-application-controller` (compares live state to Git and syncs), `argocd-redis` (caching), `argocd-dex-server` (login), plus `argocd-applicationset-controller` and `argocd-notifications-controller`.

Checked pod health before continuing, rather than assuming the install succeeded:

```
kubectl get pods -n argocd
```

`argocd-server` briefly showed `0/1` right after install (readiness probe still warming up), became `1/1` within a few minutes. All 7 pods `Running` on a 3-node cluster with 2GB per node, no resource issues encountered, worth keeping an eye on if the cluster ever gets busier.

## 2. Get the initial admin password

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 3. Access the UI from Windows (controller has no GUI)

`kubectl port-forward` only binds to the controller's own loopback interface, not reachable from the network. Reaching it from a Windows browser needs two things running at once:

**On the controller**, leave this running:

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

**From PowerShell on Windows**, a second, separate SSH connection just for tunneling:

```
ssh -L 8888:localhost:8080 mrprivii@192.168.178.134
```

Note the local port is `8888`, not `8080`. Port `8080` on Windows turned out to already be bound by another local process (confirmed with `Get-Process -Id (Get-NetTCPConnection -LocalPort 8080).OwningProcess` and `netstat -ano | findstr :8080`), not a Hyper-V/WSL port exclusion as first suspected, that was checked and ruled out with `netsh interface ipv4 show excludedportrange protocol=tcp`. Any free local port works, `8888` just happened to be free.

With both the port-forward and the SSH tunnel active, `https://localhost:8888` in a Windows browser reaches the Argo CD UI. Login: `admin` / the password from step 2.

**Confirmed working.**

## 4. Post-install security cleanup

Changed the admin password via the UI (User Info → admin → Update Password), then removed the now-unneeded initial secret:

```
kubectl -n argocd delete secret argocd-initial-admin-secret
```

## 5. First Application: deploying `forge` (a self-made app)

`forge` is a personal project (`github.com/mallegeking/forge`, "AI-Powered Gym Tracker", Next.js/TypeScript), not a public pre-built image like nginx was. Unlike nginx, this required an extra one-time step before Argo CD could do anything: turning source code into a pullable container image. Argo CD deploys manifests, it doesn't build images, that's a separate concern (normally CI's job).

### 5a. Build and push the image (manual this time, GitHub Actions planned for later)

Done from a Windows machine with Docker Desktop, since the controller doesn't have Docker installed.

```
git clone https://github.com/mallegeking/forge.git
cd forge
docker login ghcr.io -u mallegeking
docker build -t ghcr.io/mallegeking/forge:v1 --target runner .
docker push ghcr.io/mallegeking/forge:v1
```

**`docker login` gotcha:** using the actual GitHub account password fails with `denied: denied`, GitHub disabled password auth for this back in 2021, regardless of 2FA status. Needs a Personal Access Token instead (classic token, `write:packages` scope, from `github.com/settings/tokens`), pasted at the password prompt.

### 5b. Create the namespace and secret (outside of Git, deliberately)

`APP_PASSCODE` is a real secret. Committing it into Git, even a private repo, isn't good practice. Proper fix for later is Sealed Secrets (encrypts secrets so they're safe to commit), not set up yet. For now, created directly on the cluster instead, the manifests only reference it by name:

```
kubectl create namespace forge
kubectl create secret generic forge-secrets -n forge --from-literal=APP_PASSCODE='<passcode>'
```

### 5c. Kubernetes manifests

Saved under `k8s/` in the `forge` repo (the folder name is just convention, k3s uses the standard Kubernetes API, these manifests would work unmodified on any conformant cluster).

**`k8s/pvc.yaml`** — the database is a single SQLite-style file (`libsql`), needs storage that survives pod restarts:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: forge-data
  namespace: forge
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**`k8s/deployment.yaml`** — note `replicas: 1`, not 3 like nginx. A file-based database can't safely be written by pods on multiple nodes at once.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: forge
  namespace: forge
spec:
  replicas: 1
  selector:
    matchLabels:
      app: forge
  template:
    metadata:
      labels:
        app: forge
    spec:
      containers:
        - name: forge
          image: ghcr.io/mallegeking/forge:v1
          ports:
            - containerPort: 3000
          env:
            - name: TZ
              value: "Europe/Berlin"
            - name: APP_PASSCODE
              valueFrom:
                secretKeyRef:
                  name: forge-secrets
                  key: APP_PASSCODE
          volumeMounts:
            - name: data
              mountPath: /app/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: forge-data
```

**`k8s/service.yaml`**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: forge
  namespace: forge
spec:
  selector:
    app: forge
  ports:
    - port: 80
      targetPort: 3000
```

**`k8s/ingress.yaml`**:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: forge
  namespace: forge
spec:
  rules:
    - host: forge.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: forge
                port:
                  number: 80
```

Committed and pushed:

```
git add k8s/
git commit -m "Add Kubernetes manifests for deployment"
git push
```

### 5d. Register the Application with Argo CD

```
argocd app create forge \
  --repo https://github.com/mallegeking/forge.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace forge \
  --sync-policy automated \
  --self-heal
```

`--sync-policy automated` is the reconciliation loop, made real. `--self-heal` means a manual `kubectl edit` in this namespace later gets automatically reverted back to match Git.

### 5e. Troubleshooting hit: ImagePullBackOff

PVC, Service, and Ingress synced healthy immediately, Deployment got stuck:

```
Failed to pull image "ghcr.io/mallegeking/forge:v1": ... 401 Unauthorized
```

**Cause:** repo visibility and container package visibility are separate settings in GitHub. Making the `forge` repository public doesn't make the `ghcr.io/mallegeking/forge` package public, `docker push` creates packages as private by default regardless of the source repo. k3s was trying to pull with no credentials at all, and got rejected.

**Fix:** `github.com/mallegeking/forge/pkgs/container/forge` → Package settings → Danger Zone → Change visibility → Public. No Kubernetes or Argo CD changes needed, kubelet was already retrying on its own (`BackOff` loop) and succeeded on its own within a couple minutes once the image became reachable.

**Confirmed working**, `kubectl get pods -n forge` showing `1/1 Running`. Accessed by adding a hosts-file entry on the client machine (`C:\Windows\System32\drivers\etc\hosts`, edited as Administrator):

```
192.168.178.134    forge.local
```

Then `http://forge.local` reaches the app directly.

**Known issue:** entering the passcode currently throws a server error in the browser. Checked with `kubectl logs -n forge deployment/forge`, confirmed root cause: `SQLITE_ERROR: no such table: programs`, the database file exists but was never migrated. The original `docker-compose.yml` had a separate "db-tools" build target specifically for running schema migrations, that step was never part of just starting the container, and got skipped in this manual build/deploy. Everything infrastructure-side is still correct, image pull, scheduling, volume mount, Service, Ingress, secret injection, this is purely an app/deployment-process gap. Fix for the GitHub Actions redo (section 6): add a migration step, either as a Kubernetes initContainer that runs the migration before the main container starts, or baked into the image's startup command.

## 6. Still open

- **Sealed Secrets**: to be learned properly later, so real secrets can eventually live safely in Git instead of being created by hand outside it.
- **GitHub Actions CI**: automate the build-and-push step (5a) so pushing code triggers a new image automatically, rather than rebuilding by hand each time.
