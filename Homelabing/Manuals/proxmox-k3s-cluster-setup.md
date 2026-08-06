# k3s Cluster on Proxmox (1 Control-Plane + 2 Agents)

Control-plane: **400** (`k3s-ctrlr`)
Agents: **402** (`k3s-node-1`), **403** (`k3s-node-2`)
Node template: **401** (`k3s-node`, converted to a template after prep, used to spin up 402-404)
Spare, not yet joined: **404** (`k3s-node-3`)
All originally cloned as Full Clones from template 901 (see the Ubuntu 22.04 cloud-init template doc)
Reference: [k3s official requirements](https://docs.k3s.io/installation/requirements)

## 1. Clone the base VMs

Cloned from template 901:

- VMID `400`, name `k3s-ctrlr`, 2 vCPU, 2 GB RAM
- VMID `401`, name `k3s-node`, 2 vCPU, 2 GB RAM

Both set to start at boot. The node had a 15 second startup delay, so the controller gets a head start.

Official minimums are lower (server: 2 core/2GB, agent: 1 core/512MB), but keeping nodes at 2/2 leaves headroom for actual workloads later, not just the bare k3s process.

## 2. Guest agent

Installed on both, same as the original test clone:

```
sudo apt install qemu-guest-agent -y
```

Then restarted both. Baked into 401 before it became the node template, so every future node clone gets it for free. Still a manual step on the controller if it's ever rebuilt, worth baking into 901 itself at some point (see the cloud-init template doc for the two ways to do that without breaking the "never boot 901 directly" rule).

## 3. System updates

```
sudo apt update && sudo apt dist-upgrade -y
```

Cloud-init installs updates at boot by default, this was just a belt-and-suspenders check.

## 4. Static IP on the controller

Only the controller got a manually configured static IP at this stage, since it's the one address every node needs to find reliably. The node (401) was left on DHCP at this point, that turned out to matter later, see section 7.

```
cd /etc/netplan
sudo cp 50-cloud-init.yaml 50-cloud-init.yaml.bak
sudo nano 50-cloud-init.yaml
```

Correct structure (indentation matters, everything under `eth0:` at the same level, `dhcp4` set to `false` since a static address is used instead):

```yaml
network:
  version: 2
  ethernets:
    eth0:
      match:
        macaddress: "<mac address>"
      dhcp4: false
      set-name: "eth0"
      addresses: [<static-ip>/24]
      nameservers:
        addresses: [192.168.178.1]
      routes:
        - to: default
          via: 192.168.178.1
```

Test before committing:

```
sudo netplan try
```

Controller: `192.168.178.134`.

## 5. k3s-specific node prep

Much of the classic Kubernetes node prep checklist (containerd install, br_netfilter, kubeadm-style sysctl setup) does **not** apply to k3s, since it bundles its own embedded containerd and loads what it needs automatically at startup. See `kubeadm-k8s-node-prep-notes.md` for that full checklist, kept separately for a possible future full-Kubernetes install.

What's actually relevant for k3s, done on both the controller and the node before it became a template:

- **Swap:** not a hard requirement for k3s (unlike kubeadm). Checked with `free -m`, already disabled, no action needed.
- **IP forwarding:** general Linux networking requirement, not k8s-specific.

```
sudo nano /etc/sysctl.conf
```

Uncommented:

```
net.ipv4.ip_forward=1
```

- **br_netfilter:** not manually configured. K3s's own systemd service loads this automatically at startup on a normal VM (non-LXC, non-rootless), so the manual `/etc/modules-load.d/k8s.conf` step from kubeadm guides isn't needed here.
- **containerd:** not installed. K3s ships its own embedded containerd, completely separate from any system package, and ignores a system-installed one unless explicitly configured to use it. Installed accidentally while following a general Kubernetes prep guide, then removed:

```
sudo systemctl disable --now containerd
sudo apt remove --purge containerd -y
```

## 6. Firewall check

```
sudo ufw status
```

Result: `ufw` not installed/active on either node. No firewall rules needed for now. If `ufw` gets enabled later, the required ports are:

```
sudo ufw allow 6443/tcp    # k3s API, agent to server
sudo ufw allow 8472/udp    # Flannel VXLAN, node to node
sudo ufw allow 10250/tcp   # kubelet metrics/API
```

## 7. Turn the prepped node into a reusable template

Once 401 had the guest agent, updates, and k3s-specific prep done (sections 2, 3, 5, 6), it was cleaned up and converted into a template, so future k3s nodes can just be cloned instead of prepped from scratch each time.

```
sudo cloud-init clean --logs
sudo rm -rf /var/lib/cloud/instances
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id
```

Verify before shutting down:

```
ls -l /var/lib/dbus/machine-id
cat /etc/machine-id
```

`cat /etc/machine-id` should print nothing (empty file), and the `ls -l` should show a symlink pointing at `/etc/machine-id`.

```
sudo poweroff
```

Then in the Proxmox GUI: right-click 401 → Convert to Template.

**Note on network config:** `cloud-init clean` plus wiping `/var/lib/cloud/instances` resets cloud-init's per-instance state, which is exactly what's needed so a *new* clone (different instance-id) re-runs its first-boot setup instead of thinking it's already configured. Side effect worth knowing: this also means any future clone will re-render its network config from the template's Cloud-Init drive setting (DHCP), on first boot. Since 401 was never switched to a static netplan config itself (only the controller was), this wasn't an issue here, but it's the reason a static IP set by hand inside the guest wouldn't reliably survive being turned into a template. DHCP reservations, covered in the next section, avoid this problem entirely since they don't touch anything inside the guest at all.

Cloned three from the template: **402** (`k3s-node-1`), **403** (`k3s-node-2`), **404** (`k3s-node-3`, spare, not yet activated).

## 8. Fixed addresses via FritzBox DHCP reservations

Rather than editing netplan on every node clone (which would need repeating for every future node, and doesn't survive re-templating cleanly, see note above), addresses were fixed at the router instead.

In the FritzBox admin interface, for the controller and both active node clones: found each device by its MAC address (from Proxmox's Hardware tab, or `ip link show eth0` inside the guest), and ticked **"IPv4-Adresse dauerhaft zuweisen"** to lock in the address each one already had.

This also closed a gap on the controller specifically: it had a manually configured static IP (section 4), but nothing stopped the router's DHCP pool from handing that same address to some other device later. Reserving it too removes that risk.

Result, all confirmed matching:

- `k3s-ctrlr`: `192.168.178.134`
- `k3s-node-1`, `k3s-node-2`: reserved via FritzBox

## 9. Install k3s

**On the controller (`k3s-ctrlr`):**

```
curl -sfL https://get.k3s.io | sh -
```

Installs k3s as a server with defaults: embedded SQLite datastore, Flannel (VXLAN), Traefik ingress, ServiceLB, local-path storage provisioner, CoreDNS, metrics-server.

Get the join token:

```
sudo cat /var/lib/rancher/k3s/server/node-token
```

**On each agent (`k3s-node-1`, `k3s-node-2`):**

```
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.178.134:6443 K3S_TOKEN=<token> sh -
```

Since these were cloned from the already-prepped node template, no containerd/swap/sysctl/guest-agent work was needed again, that's all inherited from the template.

## 10. Verify the cluster

Run from the **controller only** (`kubectl` only has something to talk to where the API server actually runs, agents don't have a kubeconfig by default and will fail with a `localhost:8080 connection refused` error if tried there):

```
sudo k3s kubectl get nodes
```

**Confirmed working:**

```
NAME         STATUS   ROLES           AGE     VERSION
k3s-ctrlr    Ready    control-plane   22h     v1.36.2+k3s1
k3s-node-1   Ready    <none>          5m50s   v1.36.3+k3s1
k3s-node-2   Ready    <none>          5m27s   v1.36.3+k3s1
```

All three nodes `Ready`. Minor version difference between server and agents (`v1.36.2` vs `v1.36.3`) is normal for k3s and not a problem.
