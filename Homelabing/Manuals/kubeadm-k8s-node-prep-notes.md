# Node Prep Notes for a Future kubeadm / Full Kubernetes Install

These steps came up while setting up k3s, but turned out to belong to full Kubernetes (kubeadm) node prep, not k3s. K3s bundles its own embedded containerd and handles kernel module loading itself, so none of this applies there. Keeping it here for whenever a real kubeadm cluster gets set up instead.

## 1. Install and configure containerd

```
sudo apt install containerd -y
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit `/etc/containerd/config.toml`, find `runc.options`, and change:

```
SystemdCgroup = false
```

to:

```
SystemdCgroup = true
```

This aligns containerd's cgroup driver with what kubelet expects on modern systemd-based distros (cgroup v2).

## 2. Disable swap

```
free -m
```

If swap shows anything other than 0, disable it:

```
sudo swapoff -a
```

Then remove or comment out the swap line in `/etc/fstab` so it doesn't come back on reboot. kubeadm-based Kubernetes requires swap to be off.

## 3. Enable IP forwarding

```
sudo nano /etc/sysctl.conf
```

Uncomment:

```
net.ipv4.ip_forward=1
```

## 4. Load the br_netfilter kernel module

```
sudo nano /etc/modules-load.d/k8s.conf
```

Add:

```
br_netfilter
```

Then load it immediately and make the related sysctl settings take effect:

```
sudo modprobe br_netfilter
sudo nano /etc/sysctl.conf
```

Add/uncomment:

```
net.bridge.bridge-nf-call-iptables=1
```

Then apply:

```
sudo sysctl --system
```

## 5. Restart before continuing to kubeadm install

A reboot after these changes ensures the kernel module and sysctl settings are active from a clean state before running `kubeadm init` / `kubeadm join`.
