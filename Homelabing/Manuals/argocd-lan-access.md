# Argo CD over the LAN: Traefik Ingress + Pi-hole DNS

Layer: cluster access
Status: in progress, v1
Built: 12.09.2026
Cluster: k3s on `mypve` — `k3s-ctrlr` (VM 400, `192.168.178.134`), agents `192.168.178.136`, `192.168.178.137`
Hostname: `argocd.internal` (LAN only, resolved by Pi-hole CT 110)
Reference: [Argo CD ingress documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/)

Replaces the two-hop port-forward plus SSH tunnel described in `argocd-setup.md`
section 3. Depends on `pihole-lxc-fritzbox.md` for name resolution and on
`proxmox-k3s-cluster-setup.md` for the cluster itself.

---

## 1. What was wrong with the old way

Reaching the Argo CD UI meant running two things at once and keeping both alive:

```
Windows browser
   -> SSH tunnel        ssh -L 8888:localhost:8080 mrprivii@192.168.178.134
   -> controller loopback :8080
   -> kubectl port-forward svc/argocd-server 8080:443
   -> argocd-server :443
```

`kubectl port-forward` binds only to the controller's own loopback interface, so
it is not reachable from the network at all. The SSH tunnel exists purely to
carry the browser to that loopback. Three consequences:

- both hops die when the SSH session drops or the laptop sleeps
- it only works from a machine that can SSH to the controller, so never a phone
- the `argocd` CLI has the same dependency, which is the real cause of the
  "token is expired" loop noted in `n8n/n8n-cloudflare-tunnel-setup.md`

One thing the old way did get right: SSH encrypted the whole path. Any
replacement has to stay encrypted or it is a downgrade, which is why TLS is
handled explicitly in section 4 rather than left for later.

---

## 2. Why Ingress + local DNS, and not the alternatives

| Option | Why not |
|---|---|
| **Ingress + Windows hosts file** | What `forge.local` and `nginx-test.local` do today. Works, but the entry has to be repeated on every device forever and cannot be done on a phone at all. Pi-hole already exists to solve exactly this. |
| **Cloudflare Tunnel + Access** | How n8n is exposed (`n8n/n8n-cloudflare-tunnel-setup.md`). Right for n8n, wrong here: Argo CD with `--self-heal` can deploy anything to the cluster, so its UI is effectively cluster root. That stays on the LAN until there is a concrete reason it cannot. |
| **`Service type: LoadBalancer`** | k3s ServiceLB would expose it directly on the node IPs, but Traefik already holds 80 and 443, so this means a high port number to remember and no hostname. |
| **A shell alias running both hops** | Hides the friction instead of removing it. Still nothing on the phone, still dies with the session. |

The chosen path also closes an item already logged in
`Manuals/homelab-open-items.md` ("Local DNS records for homelab services") and
turns Pi-hole from an ad blocker into LAN infrastructure, which was the second
reason for building it.

---

## 3. Why `argocd.internal`

`.internal` was reserved by ICANN in 2024 for exactly this use and will never be
delegated as a public TLD. `.home.arpa` (RFC 8375) is the other formally reserved
choice.

Two names that look reasonable and are not:

- **`.local`** is reserved for mDNS. The existing `forge.local` and
  `nginx-test.local` entries only work because a hosts file is consulted before
  DNS. Served from Pi-hole, `.local` behaves unpredictably on Windows and
  Android.
- **`.home`** is not reserved by anyone. It is one of the most-queried invalid
  TLDs on the root servers and has been applied for as a gTLD, so it could start
  colliding with the LAN one day.

A subdomain of the domain already owned (`argocd.home.mallaegeking.org`) was
considered and deliberately **not** used. Worth recording *why*, because the
obvious reason is wrong: such a name would **not** have been public. Pi-hole
invents answers for the LAN, nothing is created in Cloudflare, and the name would
not resolve from outside the flat. A hostname and a service's reachability are
independent.

The real trade-off is certificates:

- a real subdomain could later get a genuine Let's Encrypt certificate via a
  DNS-01 challenge, without ever exposing the service — but every issued
  certificate is published permanently in public Certificate Transparency logs,
  so the name would announce that this homelab runs Argo CD
- `argocd.internal` can never have a publicly trusted certificate. The browser
  warning is permanent unless an internal CA (`mkcert`, `step-ca`) is set up and
  its root installed on every device

Chosen: keep LAN and public namespaces separate. Reversing this later costs one
Pi-hole record, one line of the Ingress, and a fresh login.

---

## 4. Stop Argo CD doing its own TLS

`argocd-server` serves HTTPS itself with a self-signed certificate **and**
redirects plain HTTP to HTTPS on the same hostname. Proven before changing
anything, from a throwaway pod inside the cluster:

```bash
kubectl -n argocd run curltest --rm -i --restart=Never --image=curlimages/curl:8.10.1 --quiet -- \
  -s -o /dev/null -D - http://argocd-server.argocd.svc.cluster.local/
```

```
HTTP/1.1 307 Temporary Redirect
Location: https://argocd-server.argocd.svc.cluster.local/
```

Put a plain Ingress in front of that and the result is a redirect loop: Traefik
forwards the request as HTTP, Argo CD answers 307, the browser retries at
`https://argocd.internal`, Traefik forwards it as HTTP again, forever.

The fix is to tell `argocd-server` to serve plain HTTP and let Traefik own TLS.
Argo CD ships an empty ConfigMap for flags like this, so the Deployment does not
need editing:

```bash
kubectl -n argocd patch cm argocd-cmd-params-cm --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deploy argocd-server
kubectl -n argocd rollout status deploy argocd-server
```

The restart is required. The ConfigMap is read at startup, not watched.

Same check afterwards:

```
HTTP/1.1 200 OK
```

**"insecure" is a misleading name.** It means "this process does not terminate
TLS", not "this traffic is unencrypted". The encrypted hop moves to Traefik; the
only plaintext left is Traefik to the pod, inside the cluster network. The wire
between browser and cluster stays encrypted.

Note this also changes the old access path: with TLS off on the pod, the
port-forward fallback in `argocd-setup.md` is now `http://localhost:8888`, not
`https://`.

---

## 5. The Ingress

Two objects. Applied by hand first to prove them, moved into Git afterwards
(section 9).

```yaml
# Redirects any plain-HTTP request on this router to HTTPS, so the Argo CD
# login page is never served in cleartext on the LAN.
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: argocd
spec:
  redirectScheme:
    scheme: https
    permanent: true
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    # <namespace>-<middleware name>@kubernetescrd is Traefik's naming scheme for
    # a Middleware defined as a Kubernetes CRD.
    traefik.ingress.kubernetes.io/router.middlewares: argocd-redirect-https@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: argocd.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
  # No secretName: Traefik serves its own default self-signed certificate.
  # Present only so Traefik opens an HTTPS router for this host at all.
  tls:
    - hosts:
        - argocd.internal
```

Points worth understanding:

- **`port: 80` on a Service that also offers 443.** Both Service ports forward to
  the same container port. With `server.insecure` set, 443 no longer speaks TLS,
  so using 80 is the honest choice.
- **The empty `tls:` block.** Without it Traefik never opens an HTTPS router for
  this host and only answers on port 80. With it, and no `secretName`, Traefik
  falls back to the default self-signed certificate it generates on startup. This
  is the line to change when a real certificate exists.
- **The middleware is what keeps the password off the wire.** Without it the
  login page would also answer on plain HTTP, which would be worse than the SSH
  tunnel it replaces.
- **CRD API group.** This cluster runs Traefik 3.7.4, so middlewares are
  `traefik.io/v1alpha1`. Older guides use `traefik.containo.us/v1alpha1`, which
  no longer exists.

```bash
kubectl apply -f ingress.yaml
kubectl -n argocd get ingress argocd-server
```

```
NAME            CLASS     HOSTS             ADDRESS                                           PORTS
argocd-server   traefik   argocd.internal   192.168.178.134,192.168.178.136,192.168.178.137   80, 443
```

**All three node IPs appear, and that is not cosmetic.** k3s's ServiceLB binds
ports 80 and 443 on every node, so any of the three serves this ingress. The DNS
record in the next section points at `192.168.178.134` for simplicity, but it
could point at any node, or at all three for crude failover.

---

## 6. The DNS record

Pi-hole (CT 110) is what turns the hostname into an address for the whole LAN,
with no hosts file on any device. This is the step that makes the whole thing
worth doing, and it is 30 seconds of clicking.

**Pi-hole admin > Settings > Local DNS Records**

| Domain | IP |
|---|---|
| `argocd.internal` | `192.168.178.134` |

**A record, not CNAME.** The Local CNAME Records box next to it stays empty. A
CNAME aliases one *name* to another *name* (`argo.internal` -> `argocd.internal`).
What is needed here is a name pointing at an *address*, which is an A record.
CNAMEs become useful later, when several names should follow one canonical host.

---

## 7. Verification

### DNS

From Windows, flushing first so a stale cached answer cannot fake a success:

```powershell
ipconfig /flushdns
nslookup argocd.internal
```

The useful part of the answer is not the address, it is **which server replied**.
If the `Server:` line shows `192.168.178.1`, the machine is still asking the
FritzBox directly and never reached Pi-hole at all — reconnect the wifi to pull a
fresh DHCP lease (same caveat as `pihole-lxc-fritzbox.md` section 4).

**Observed here: Pi-hole answered on its IPv6 address**, not `192.168.178.9`.
That is correct and is worth unpacking, because it is two independent things that
look like one:

- **The transport was IPv6.** The FritzBox advertises Pi-hole's ULA as the DNSv6
  server over Router Advertisement (`pihole-lxc-fritzbox.md` section 4), and
  Windows prefers IPv6 for the resolver when both are offered. So the *question*
  travelled over IPv6.
- **The answer was still an A record**, `192.168.178.134` — an IPv4 address,
  because that is what was entered in Pi-hole.

The protocol you talk to a resolver over and the kind of record it hands back are
unrelated. A resolver reached over IPv6 happily returns IPv4 addresses.

Incidentally this is a live confirmation of check 3 in the Pi-hole doc's own
verification section: the IPv6 path works end to end.

### Browser

`https://argocd.internal` — type the scheme explicitly, or the browser treats a
dotted word as a search term.

**A certificate warning here is the expected result, not a failure.** Traefik is
serving the self-signed certificate it generated at startup (the issuer reads as
Traefik's default cert), because a private hostname like `.internal` can never
have a publicly trusted certificate. This was the accepted trade-off in section
3. Click through and the Argo CD login appears.

**Confirmed working** from Windows and from a phone on the same wifi. The phone
is the real proof: it was never reachable under the old port-forward setup, and
there is no hosts file on it to edit.

### What replaced what

```
before:  browser -> SSH tunnel -> controller loopback -> port-forward -> argocd-server
after:   browser -> Pi-hole (name) -> Traefik on any node -> argocd-server
```

Nothing has to be kept running by hand any more.

---

## 8. The controller cannot resolve it (open decision)

Predicted before testing, and confirmed:

```bash
getent hosts argocd.internal      # on k3s-ctrlr: no output
```

The cluster's own control node cannot resolve the name that now works on every
phone in the flat. The reason is a collision between two decisions made in
separate docs, neither wrong on its own:

- `proxmox-k3s-cluster-setup.md` section 4 gave the controller a **static netplan
  config** with `nameservers: [192.168.178.1]` — the FritzBox.
- `pihole-lxc-fritzbox.md` section 1 deliberately **did not** make Pi-hole the
  FritzBox's upstream, because that plus conditional forwarding creates a DNS
  loop.

So the controller asks a resolver that has never heard of `argocd.internal`. The
agents (`192.168.178.136`, `192.168.178.137`) take DNS from a DHCP lease, so they
resolve it fine. The controller is the odd one out precisely *because* it was
given a static config.

This matters because the `argocd` CLI runs on the controller, so
`argocd login argocd.internal` cannot work there yet.

Two fixes, not yet chosen:

| Option | Gain | Cost |
|---|---|---|
| **Point netplan at `192.168.178.9`** | Uniform DNS across the whole cluster. Every future `*.internal` name works everywhere, including inside pods — k3s CoreDNS forwards anything non-cluster to the host's resolver, so cluster workloads would resolve local names and get ad blocking on egress too. | Deepens the dependency on CT 110. If Pi-hole is down, the control node cannot resolve `ghcr.io` either, so image pulls fail. That is the bootstrap loop described in `pihole-lxc-fritzbox.md` section 2, reaching one node further. |
| **One `/etc/hosts` entry on the controller** | Zero new dependency. Keeps one machine with an independent resolver as a deliberate escape hatch when Pi-hole is down. | Reintroduces exactly the hosts-file workaround this whole build was meant to retire, and has to be repeated for every future service name. |

Not decided yet. Adding the FritzBox as a *secondary* resolver is not one of the
options: the Pi-hole doc rules that out for clients (round-robin silently
bypasses blocking), and it is worse for a server, where it would make local names
resolve only intermittently — the hardest possible failure to debug.

---

## 9. Moving it into Git

Not done yet. The Ingress and Middleware currently exist only because
`kubectl apply` was run by hand, which means a cluster rebuild loses them and
there is no record of why they exist — the same gap Sealed Secrets is meant to
close for secrets.

Planned: commit both objects to the `homelab-apps` repo under `argocd/`, then
register them as an Argo CD Application, the same pattern as n8n.

The interesting part is that Argo CD would then manage its own front door.
Consequences to think through before doing it:

- the resources already exist without Argo CD's tracking labels, so the first
  sync **adopts** them rather than creating them
- with `--self-heal`, a bad commit to this manifest breaks the UI used to watch
  the sync that broke it
- the fallback if that happens is still `kubectl port-forward`, which is why
  section 4's note about it now being `http://` and not `https://` matters

---

## 10. Deliberately **not** done

- **No Cloudflare Tunnel.** Argo CD with `--self-heal` can deploy anything to the
  cluster, so its UI is effectively cluster root. n8n is exposed publicly
  (`n8n/n8n-cloudflare-tunnel-setup.md`) because the blast radius there is one
  automation tool. This stays on the LAN.
- **No real certificate.** Impossible for a `.internal` name from any public CA.
  The routes out are an internal CA (`mkcert`, `step-ca`) with its root installed
  on every device, or switching to a subdomain of the owned domain and using a
  DNS-01 challenge. Both are projects, neither is needed to use the UI.
- **No HTTP-only ingress.** The redirect middleware is not decoration. Without it
  the login page answers on plain HTTP, which would be a downgrade from the SSH
  tunnel this replaces.
- **`forge.local` and `nginx-test.local` left alone.** They still rely on Windows
  hosts-file entries. Converting them to Pi-hole records is the obvious next
  application of this pattern, but changing them was out of scope here.

---

## 11. Open items

Tracked in the Notion **Threads** database (Category: Homelab), which replaced
`homelab-open-items.md` as the live backlog. Both carry a Review date of
14.09.2026.

- **Decide the k3s controller's DNS resolver** (Decision) — section 8
- **Move the Argo CD Ingress into `homelab-apps`** (Build) — section 9

Lower priority, same database: convert `forge.local` and `nginx-test.local` to
Pi-hole records, and the existing "HTTPS inside the local network" thread, which
is now the certificate-warning problem.

---

## 12. Everything that was changed, and the exact commands

Four changes total: three on the cluster, one in Pi-hole. Nothing was changed on
the Proxmox host, the FritzBox, or any client machine.

| Change | Why it was needed |
|---|---|
| `server.insecure: "true"` in `argocd-cmd-params-cm` + rollout restart | `argocd-server` answered plain HTTP with a `307` to HTTPS on the same host, so a plain Ingress would loop forever |
| Traefik `Middleware` `redirect-https` | Without it the login page also answers on plain HTTP, a downgrade from the SSH tunnel being replaced |
| `Ingress/argocd-server`, host `argocd.internal`, backend port 80 | Port 443 on that Service no longer speaks TLS, so 80 is the honest target |
| `tls:` block with no `secretName` | Makes Traefik open an HTTPS router at all; with no secret it serves its own self-signed certificate |
| Pi-hole Local DNS Record `argocd.internal` -> `192.168.178.134` | Turns the hostname into an address for every device on the LAN |

### Run on the controller (`k3s-ctrlr`)

These were executed over SSH from WSL, each prefixed with
`export KUBECONFIG=~/.kube/config`. That prefix is needed because a
non-interactive SSH command does not read `~/.bashrc`, where
`proxmox-k3s-cluster-setup.md` section 11 put the `KUBECONFIG` export. Running
them in an interactive session on the controller does not need it.

**Inspect first, change nothing:**

```bash
kubectl get nodes
kubectl -n argocd get svc argocd-server
kubectl get ingressclass
kubectl -n kube-system get svc traefik
kubectl get ingress -A
kubectl -n argocd get cm argocd-cmd-params-cm -o yaml
kubectl get crd | grep -i traefik
kubectl -n kube-system get deploy traefik -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Prove the redirect loop before fixing it:**

```bash
kubectl -n argocd run curltest --rm -i --restart=Never --image=curlimages/curl:8.10.1 --quiet --   -s -o /dev/null -D - http://argocd-server.argocd.svc.cluster.local/
```

**Change 1 — hand TLS to Traefik:**

```bash
kubectl -n argocd patch cm argocd-cmd-params-cm --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deploy argocd-server
kubectl -n argocd rollout status deploy argocd-server --timeout=120s
```

**Re-run the same probe** (`307` before, `200 OK` after):

```bash
kubectl -n argocd run curltest2 --rm -i --restart=Never --image=curlimages/curl:8.10.1 --quiet --   -s -o /dev/null -D - http://argocd-server.argocd.svc.cluster.local/
```

**Changes 2 and 3 — the Ingress and Middleware** (file contents in section 5):

```bash
mkdir -p ~/manifests/argocd
# write ~/manifests/argocd/ingress.yaml
kubectl apply -f ~/manifests/argocd/ingress.yaml
kubectl -n argocd get ingress argocd-server
```

### Run from a machine on the LAN, not from the cluster

The point is proving reachability from outside, the same reasoning as the nginx
test in `proxmox-k3s-cluster-setup.md` section 12. `--resolve` stands in for DNS,
so the ingress can be tested before the Pi-hole record exists.

```bash
# expect HTTP/2 200
curl -sk -o /dev/null -D - --resolve argocd.internal:443:192.168.178.134 https://argocd.internal/

# expect 301 to https://argocd.internal/
curl -s -o /dev/null -D - --resolve argocd.internal:80:192.168.178.134 http://argocd.internal/

# inspect the TLS handshake and certificate
curl -skv --resolve argocd.internal:443:192.168.178.134 https://argocd.internal/
```

### Change 4 and the real verification

Pi-hole record added through the web UI (section 6), then from Windows:

```powershell
ipconfig /flushdns
nslookup argocd.internal
```

And on the controller, the check that confirmed the gap in section 8:

```bash
getent hosts argocd.internal      # no output
```

---

## 13. Rollback

Nothing here is one-way. To put everything back exactly as it was:

```bash
kubectl -n argocd delete ingress argocd-server
kubectl -n argocd delete middleware redirect-https
kubectl -n argocd patch cm argocd-cmd-params-cm --type=json -p '[{"op":"remove","path":"/data/server.insecure"}]'
kubectl -n argocd rollout restart deploy argocd-server
```

Then delete the `argocd.internal` record in Pi-hole. The old port-forward plus
SSH tunnel path from `argocd-setup.md` section 3 works again, over `https://`
once `server.insecure` is gone.
