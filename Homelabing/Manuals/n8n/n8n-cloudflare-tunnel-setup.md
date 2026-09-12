# n8n on k3s via Cloudflare Tunnel

Deployed to the k3s cluster through Argo CD, exposed publicly at `https://n8n.mallaegeking.org` via a Cloudflare Tunnel, gated behind Cloudflare Access.

Manifests live in the `homelab-apps` repo (`github.com/mallegeking/homelab-apps`), under `n8n/`. That repo holds third-party apps deployed to the cluster, separate from own-code repos like `forge` which keep manifests alongside their source.

## Why a tunnel instead of Ingress

The nginx and forge deployments used Traefik Ingress with hosts-file entries pointing at the controller's LAN IP. That works, but has two limits: it only works on the LAN, and editing hosts files doesn't extend to phones.

A Cloudflare Tunnel solves both. `cloudflared` runs as a pod inside the cluster and makes an outbound connection to Cloudflare, traffic comes back down through it. No port forwarding on the router, nothing exposed directly to the internet, real HTTPS, and it works from any device anywhere. It also makes webhook-triggered workflows possible, since external services can actually reach the instance.

Traefik is not in the path for n8n at all. No Ingress manifest is used here, cloudflared talks straight to the Service.

## 1. Generate the encryption key first

`N8N_ENCRYPTION_KEY` encrypts every credential stored in n8n. If it's lost or changes, every saved credential becomes permanently undecryptable, with no recovery path. It has to be set deliberately and kept stable for the life of the instance.

```
openssl rand -hex 24
```

Saved to KeePass immediately, before anything else.

## 2. Create the Cloudflare Tunnel

In the Cloudflare dashboard: **Zero Trust → Networks → Tunnels → Create a tunnel → Cloudflared**, named `homelab-k3s`.

On the install screen, ignore the install commands shown (those are for bare metal/Docker). Copy only the **token** from the command, the long string after `--token`.

Then add the route. In the current UI this is under the **Published applications** tab (older docs call this "Public hostname"):

- Subdomain: `n8n`
- Domain: `mallaegeking.org`
- Type: **HTTP** (not HTTPS)
- URL: `n8n.n8n.svc.cluster.local:80`

That URL is Kubernetes internal DNS, following the pattern `<service>.<namespace>.svc.cluster.local`. So: service `n8n`, in namespace `n8n`. Since cloudflared runs as a pod inside the cluster, it resolves this directly. Plain HTTP is fine on that hop, it's internal cluster traffic, and the tunnel itself is encrypted. Cloudflare still serves real HTTPS to the outside world.

Cloudflare creates the DNS record automatically.

## 3. Create the namespace and secrets

Secrets are created directly on the cluster, not committed to Git. The manifests reference them by name only.

```
kubectl create namespace n8n
kubectl create secret generic n8n-secrets -n n8n --from-literal=N8N_ENCRYPTION_KEY='<openssl key from step 1>'
kubectl create secret generic cloudflared-token -n n8n --from-literal=TUNNEL_TOKEN='<tunnel token from step 2>'
```

## 4. Manifests

Three files in `homelab-apps/n8n/`. No ingress.yaml.

**`pvc.yaml`** — n8n stores its SQLite database and config in `/home/node/.n8n`, this keeps it across pod restarts:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: n8n-data
  namespace: n8n
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

**`deployment.yaml`**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n
  namespace: n8n
spec:
  replicas: 1
  selector:
    matchLabels:
      app: n8n
  template:
    metadata:
      labels:
        app: n8n
    spec:
      securityContext:
        fsGroup: 1000
      containers:
        - name: n8n
          image: docker.io/n8nio/n8n:latest
          ports:
            - containerPort: 5678
          env:
            - name: N8N_ENCRYPTION_KEY
              valueFrom:
                secretKeyRef:
                  name: n8n-secrets
                  key: N8N_ENCRYPTION_KEY
            - name: N8N_HOST
              value: "n8n.mallaegeking.org"
            - name: N8N_PORT
              value: "5678"
            - name: N8N_PROTOCOL
              value: "https"
            - name: N8N_WEBHOOK_URL
              value: "https://n8n.mallaegeking.org/"
            - name: N8N_PROXY_HOPS
              value: "1"
            - name: GENERIC_TIMEZONE
              value: "Europe/Berlin"
            - name: TZ
              value: "Europe/Berlin"
          volumeMounts:
            - name: data
              mountPath: /home/node/.n8n
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: n8n-data
```

Notes on the non-obvious settings:

- `fsGroup: 1000` — n8n runs as a non-root user and needs write permission on the mounted volume. Without this the pod fails on startup.
- `N8N_PROXY_HOPS: "1"` — n8n sits behind a proxy (the tunnel), without this it builds incorrect URLs.
- `N8N_PROTOCOL: "https"` and `N8N_WEBHOOK_URL` — these control the URLs n8n generates for OAuth callbacks and webhooks. Set to `http` initially by mistake, which still loads the site (Cloudflare serves HTTPS regardless) but would break OAuth and webhook integrations later.
- `N8N_WEBHOOK_URL` not `WEBHOOK_URL` — the latter is deprecated, n8n logs a warning about it on startup.

**`cloudflared.yaml`**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: n8n
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          args:
            - tunnel
            - --no-autoupdate
            - run
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: cloudflared-token
                  key: TUNNEL_TOKEN
```

cloudflared pulls its routing config down from Cloudflare using the token, nothing about the hostname or service is hardcoded here.

**`service.yaml`**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: n8n
  namespace: n8n
spec:
  selector:
    app: n8n
  ports:
    - port: 80
      targetPort: 5678
```

## 5. Register with Argo CD

```
argocd app create n8n \
  --repo https://github.com/mallegeking/homelab-apps.git \
  --path n8n \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace n8n \
  --sync-policy automated \
  --self-heal
```

After this, changes to the manifests only need a `git push`. Argo CD picks them up and applies them within a few minutes, no `kubectl` involved.

## 6. Cloudflare Access (login gate)

Without this, anyone with the URL reaches the n8n login page. Access puts a Cloudflare login in front of it, so only approved emails reach the app at all.

Zero Trust → **Access controls → Applications → Add an application → Self-hosted and private**.

In the current UI, pick the **Public DNS** destination tab, `n8n.mallaegeking.org` is a public hostname served through the tunnel, reachable from any browser. "Private destinations" is for resources only reachable via the Cloudflare One client, which isn't the setup here.

Then: name the application, add the hostname, and create a policy with Action **Allow**, Include → **Emails** → the allowed addresses.

**Gotcha hit here:** the application itself didn't save on the first attempt, leaving the policy created but attached to nothing. The dashboard looked correct, but the site was still wide open. Creating the application properly fixed it.

**Always verify in a private/incognito window.** An unauthenticated browser should hit the Cloudflare Access login screen ("Log in to n8n", email + login code), not n8n itself. This is the only reliable check, an already-logged-in browser will pass through either way and tells you nothing.

**Confirmed working**, unauthenticated requests now hit the Access login gate.

**Known limitation for later:** if workflows use external webhooks (a service calling in from the internet), those webhook paths need an Access **bypass** policy. Otherwise the external service gets blocked by the login gate the same as a browser would.

## 7. Verification

```
kubectl get pods -n n8n
```

Both `n8n` and `cloudflared` showing `1/1 Running`.

```
kubectl logs -n n8n deployment/cloudflared
kubectl logs -n n8n deployment/n8n
```

**Confirmed working.** cloudflared registered four connections to Cloudflare edge locations (dus01, fra14, fra12) with all connectivity pre-checks passing, and received its config from Cloudflare mapping `n8n.mallaegeking.org` to the internal service. n8n ran its full database migration set cleanly on first start and came up on version 2.33.7. `https://n8n.mallaegeking.org` loads.

## 8. Upgrading n8n

### Pin the version, don't use `:latest`

The original manifest used `docker.io/n8nio/n8n:latest`. That looks like it should auto-update, but it doesn't: the tag string in Git never changes, so Argo CD sees no diff and never syncs, and the pod keeps running whatever version it first pulled. The instance sat four versions behind without any indication anything was stale.

Pinning an explicit version fixes this and makes upgrades auditable:

```yaml
          image: docker.io/n8nio/n8n:2.37.9
```

With a pinned tag, `git log` on that line is the upgrade history, and rollback is `git revert`. With `latest`, there's no record of what was running before, so there's nothing to roll back to.

An empty commit doesn't help either. Argo CD will notice the new revision and re-sync, but the manifests are byte-identical, so the pod spec doesn't change and nothing restarts.

### Order of operations

**1. Check the release notes** between the current version and the target. Patch releases rarely need manual steps, major version jumps sometimes do.

**2. Back up the data volume, before pushing anything.** Upgrades run database migrations and those are one-way, rolling the image tag back does not undo them. This command runs against the *running* pod, so it has to happen before Argo CD rolls the deployment. Once the old pod is gone there's nothing left to snapshot.

```
kubectl exec -n n8n deployment/n8n -- tar czf - -C /home/node/.n8n . > n8n-backup-$(date +%F).tar.gz
```

**3. Edit the image tag, commit, push.** Argo CD picks it up and rolls the pod within a few minutes.

**4. Verify:**

```
kubectl get pods -n n8n
kubectl get pods -n n8n -o jsonpath='{.items[*].spec.containers[*].image}'
kubectl logs -n n8n deployment/n8n
```

The logs show which migrations ran and confirm the version at startup.

### Notes

- **A `502 Bad Gateway` during the roll is expected.** cloudflared can't reach the service while the old pod is terminating and the new one is still pulling and running migrations. The image pull alone took about 3 minutes here.
- **Skipping intermediate patch versions is fine.** Migrations are cumulative, n8n runs whichever haven't run yet, in order. Going 2.37.7 → 2.37.9 directly runs the same migrations as stepping through 2.37.8 first. Sequential upgrades matter for major version jumps or large gaps, not patch hops.
- **n8n will usually show "one version behind."** It releases frequently enough that chasing zero isn't worth it. Upgrade when there's a reason: a needed fix, a wanted feature, or a security advisory.
- **`cloudflared` is still pinned to `:latest`** in this setup, same pitfall applies. Less urgent since it holds no data, but worth pinning eventually.

## Notes

- **Python task runner warning on startup** is expected, not an error. The image ships without Python 3, so Python-based Code nodes won't work. JavaScript Code nodes work normally.
- **Deprecation warnings** in the startup log list env vars whose defaults change in future versions. Worth revisiting on a major upgrade, none are breaking now.

## Troubleshooting notes from this setup

- **`argocd` CLI: "token is expired"** — the CLI session expires. Re-run `argocd login localhost:8080 --username admin --password '<password>' --insecure`. Requires the port-forward to `argocd-server` to be running.
- **`kubectl port-forward` "address already in use"** — means a port-forward is already running (likely in another SSH session), not that something is broken. Check with `ss -tlnp | grep 8080`.
- **`git push` "Password authentication is not supported"** — GitHub requires a Personal Access Token instead of an account password. `git config --global credential.helper store` saves it after the first successful push.
- **Heredoc file creation writing to the wrong path** — check the working directory first. Running `cat > n8n/pvc.yaml` while already inside the `n8n/` folder tries to write to `n8n/n8n/pvc.yaml` and fails.
