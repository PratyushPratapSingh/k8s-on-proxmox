# k8s-on-proxmox Quick Start Installer
For Proxmox server, k8 home lab setup

This guide is organized as a copy-paste installer checklist, but it still includes the important notes that explain why each command is needed.

## Prerequisites

Set your router DHCP scope so you have a static IP pool for the cluster.

Example:
- DHCP pool: `192.168.1.2 - 192.168.1.99`
- Static IPs for Kubernetes: `192.168.1.100 - 192.168.1.254`

The cluster machines should use static IPs so that the master and workers keep stable addresses.

---

## Quick Start Checklist

### 1) Create the VMs in Proxmox

Run these commands in the Proxmox shell.

#### Master Node (VM ID 191)

```bash
qm create 191 \
  --name k8s-master \
  --agent 1 \
  --balloon 0 \
  --cores 4 \
  --sockets 1 \
  --cpu host \
  --memory 4096 \
  --numa 0 \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0,firewall=1 \
  --ide2 local:iso/ubuntu-26.04-live-server-amd64.iso,media=cdrom \
  --boot "order=scsi0;ide2;net0" \
  --scsi0 local-lvm:50,discard=on,iothread=1,ssd=1
```

#### Worker Node 1 (VM ID 192)

```bash
qm create 192 \
  --name k8s-worker1 \
  --agent 1 \
  --balloon 0 \
  --cores 2 \
  --sockets 1 \
  --cpu host \
  --memory 2048 \
  --numa 0 \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0,firewall=1 \
  --ide2 local:iso/ubuntu-26.04-live-server-amd64.iso,media=cdrom \
  --boot "order=scsi0;ide2;net0" \
  --scsi0 local-lvm:50,discard=on,iothread=1,ssd=1
```

#### Worker Node 2 (VM ID 193)

```bash
qm create 193 \
  --name k8s-worker2 \
  --agent 1 \
  --balloon 0 \
  --cores 2 \
  --sockets 1 \
  --cpu host \
  --memory 2048 \
  --numa 0 \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0,firewall=1 \
  --ide2 local:iso/ubuntu-26.04-live-server-amd64.iso,media=cdrom \
  --boot "order=scsi0;ide2;net0" \
  --scsi0 local-lvm:50,discard=on,iothread=1,ssd=1
```

> Note: If you use custom storage instead of `local-lvm`, replace the disk `scsi0` value with your storage location.

---

### 2) Prepare each VM (run on ALL VMs)

Boot each VM and then SSH into it.

#### Update OS and reboot

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

#### Install qemu guest agent

```bash
sudo apt install qemu-guest-agent -y
```

#### Disable swap

Kubelet expects swap to be off. If swap remains enabled, Kubernetes may behave unpredictably.

```bash
sudo swapoff -a
sudo nano /etc/fstab
```

Comment out the line containing `swap`, then save and exit.

Example:
```bash
# /swap.img none swap sw 0 0
```

#### Configure hostname and hosts

```bash
cat /etc/hostname
cat /etc/hosts
```

Make sure each VM has the correct hostname and entries for the cluster nodes.

#### Load kernel modules

These modules are required for overlay networking and bridged traffic.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

#### Configure sysctl values

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

#### Install containerd

Containerd is the container runtime used by Kubernetes.

```bash
sudo apt install containerd -y

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

The service should show `active (running)`.

#### Install Kubernetes packages

This installs `kubelet`, `kubeadm`, and `kubectl`.

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Why `apt-mark hold` matters:
- `kubelet`, `kubeadm`, and `kubectl` must stay at the same version
- If a later `apt upgrade` updates them separately, Kubernetes can break
- `hold` prevents accidental mismatched upgrades

Example:

```bash
sudo apt-mark hold kubelet kubeadm kubectl

# Later, if you want to allow upgrades again:
sudo apt-mark unhold kubelet kubeadm kubectl
```

#### IPv6 troubleshooting note

If you see errors like:
- `Failed to fetch https://pkgs.k8s.io/... 403 Forbidden`
- or an IPv6 address such as `2600:9000:...` appears in the error output

This usually means IPv6 is failing or blocked on your network. Force IPv4 for package installs.

```bash
sudo apt -o Acquire::ForceIPv4=true update
sudo apt -o Acquire::ForceIPv4=true install -y apt-transport-https ca-certificates curl gpg
```

Then retry the signing key and repo:

```bash
curl -fsSL --ipv4 https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt -o Acquire::ForceIPv4=true update
```

If your network has no working IPv6, you can also disable IPv6 on the VM:

```bash
sudo nano /etc/sysctl.d/99-disable-ipv6.conf
```

Add:

```bash
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

Apply:

```bash
sudo sysctl -p /etc/sysctl.d/99-disable-ipv6.conf
```

---

### 3) Initialize the Kubernetes master node

Run this on the master node only:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Why `--pod-network-cidr=10.244.0.0/16`?
- This is the IP range used by the pod network inside the cluster
- It is separate from your LAN network such as `192.168.1.0/24`
- It must not overlap with your home network or other internal networks
- Flannel is the CNI we use, and it expects this default range

This tells Kubernetes:
- create the control plane for the cluster
- assign pod IPs from `10.244.0.0/16`
- use Flannel later to route pod traffic between nodes

#### Configure kubectl access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Install Flannel CNI

Flannel is the pod networking plugin that creates the virtual overlay network for Kubernetes pods. Without it, pods cannot talk to each other.

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

This installs the Flannel daemonset that assigns pod IPs and routes packet traffic between nodes.

#### Check the control plane

```bash
kubectl get nodes
kubectl cluster-info
kubectl get pods -n kube-system
kubectl get pods -n kube-flannel
```

You want to see:
- the node in `Ready` status
- control-plane pods in `Running`
- Flannel pods in `Running`

If `kubelet` is `inactive`, you likely have not run `kubeadm init` yet.

---

### 4) Join the worker nodes

After the master completes initialization, the command output will include a join command like this:

```bash
sudo kubeadm join 192.168.1.191:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Run that exact command on each worker node.

Then verify from the master node:

```bash
kubectl get nodes -o wide
```

The output should show all nodes as `Ready`.

---

### 5) Verify the cluster

Run on the master node:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get nodes -o wide
```

You should see something like:

```text
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   5m    v1.32.0
k8s-worker1   Ready    <none>          3m    v1.32.0
k8s-worker2   Ready    <none>          3m    v1.32.0
```

Your cluster setup is complete when:
- `kubelet` is `active (running)`
- all nodes are `Ready`
- Flannel pods are `Running`
- all control-plane pods are `Running`

---

### 6) Optional: Install MetalLB for LoadBalancer services

MetalLB enables `LoadBalancer` services in a home lab environment.

#### Update kube-proxy config

```bash
kubectl edit configmap -n kube-system kube-proxy
```

In the `ipvs` section, change:

```yaml
strictARP: false
```

to:

```yaml
strictARP: true
```

Save and exit with `:wq` in vim.

#### Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
kubectl get pods -n metallb-system
```

Make sure all MetalLB pods are `Running`.

#### Create IP pool

```bash
cat <<EOF > proxmox-ip-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: marek-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.245
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: layer2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - marek-pool
EOF

kubectl apply -f proxmox-ip-pool.yaml
```

#### Test LoadBalancer service

```bash
cat <<EOF > nginx-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f nginx-test.yaml
kubectl get svc nginx-service
```

You should see an `EXTERNAL-IP` such as `192.168.1.240`.

Open it in a browser:

```text
http://192.168.1.240:80
```

You should see the Nginx default page.

---

## Troubleshooting

### kubelet is inactive

This usually means the cluster was not initialized yet.

Run:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Then configure kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 403 Forbidden while fetching `pkgs.k8s.io`

This typically means the package download path is blocked or IPv6 is failing.

Retry with force IPv4:

```bash
sudo apt -o Acquire::ForceIPv4=true update
sudo apt -o Acquire::ForceIPv4=true install -y apt-transport-https ca-certificates curl gpg
```

Then retry repository setup:

```bash
curl -fsSL --ipv4 https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt -o Acquire::ForceIPv4=true update
```

### Pods stuck in `Pending` or `CrashLoopBackOff`

Check the pod and node events:

```bash
kubectl describe pod <pod-name> -n kube-system
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Check kubelet logs:

```bash
sudo journalctl -u kubelet -n 100 --no-pager
```

---

## One-Sentence Summary of the Key Concepts

- `kubeadm init` sets up the Kubernetes control plane on the master node.
- `--pod-network-cidr=10.244.0.0/16` defines the range of IPs used by pods.
- Flannel is the pod network plugin that makes pods on different nodes talk to each other.
- `kubelet` runs workloads on each node.
- `kubeadm join` connects worker nodes to the master.
- `apt-mark hold kubelet kubeadm kubectl` keeps versions in sync and prevents accidental cluster breakage.

---

## Final Notes

This setup is designed for a small home-lab Kubernetes cluster built on Proxmox. The main idea is:

- All VMs prepare the OS and install Kubernetes packages
- Master node initializes the cluster and installs the CNI
- Workers join the cluster using the token generated by the master
- Flannel provides the pod network so containers can communicate

Once this is working, you can deploy apps, services, ingress, and any Kubernetes workloads you need.

Enjoy your Kubernetes home lab.
