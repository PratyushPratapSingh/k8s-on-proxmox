# k8s-on-proxmox
For Proxmox server, k8 home lab setup

## Prerequisites

First we need to change the DHCP scope on home router so we can have a range of IP addresses that we can assign statically.
My DHCP scope is `192.168.1.2 - 192.168.1.99`,
so any addresses from `192.168.1.100` to `192.168.1.254` can be assigned statically.

## Create the VMs

We can now log on to Proxmox and create 3 Linux Virtual Machines:
- 1 Kubernetes Master Node
- 2 Worker Nodes (you can create more if you wish; this is just an example)

The master node acts as the cluster administrator, while the worker nodes run the containers.

Any Linux kernel-based system can run Kubernetes, but the easiest way to follow this guide is to use Ubuntu or another Debian-based Linux OS.

I will go for Ubuntu 26.04 LTS. Go to the Ubuntu download page and download the live server ISO.

Scroll down to the live server ISO file and click "copy link address". Then click "query link" in Proxmox and download it.

Now (optionally) run "Create VM". You don't need to install the operating system yet; it's only to see the config file. I will create that VM with ID 190.

In the Proxmox console, run:

```bash
qm config 190
```

This will show the output of `/etc/pve/qemu-server/190.conf`.
Run `man qm` to combine that output with the instructions for the `qm` command.

Now let's create 3 virtual machines based on that information. Note that the master node might need a bit more resources than the worker nodes.

### Master node (VM ID 191)

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
  --scsi0 transcend:50,discard=on,iothread=1,ssd=1
```

### Worker node 1 (VM ID 192)

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
  --scsi0 transcend:50,discard=on,iothread=1,ssd=1
```

### Worker node 2 (VM ID 193)

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
  --scsi0 transcend:50,discard=on,iothread=1,ssd=1
```

> Note: This is true for my setup with a Transcend SSD used as external storage. If you run the default Proxmox setup, the `qm` code might look more like this:

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

Use the output of your config file or go through the manual process of VM creation. Either is fine.

## Quick node setup map

Before you run any Kubernetes commands, remember this simple rule:

- Run on all VMs: system updates, networking setup, containerd, kubelet/kubeadm/kubectl installation
- Run on the master node only: `kubeadm init`, `kubectl` config, CNI installation
- Run on each worker node only: `kubeadm join` using the master-generated token

In other words:

```text
All VMs -> prepare the OS and install Kubernetes packages
Master  -> initialize cluster and install CNI
Workers -> join the cluster
```

This is the most important distinction in this guide. The master does not need the worker join command, and the workers do not run `kubeadm init`.

## Initial VM setup

Now start each VM and configure the hostname and static IP address.

If you wonder whether we could use cloud images or templates, yes, we could. Or we could set up one instance and clone it. But those solutions are more confusing, and for 3 VMs they are not even [..]

For cloned images, you would need to remove machine IDs, re-provision SSH keys, and more. Installing each instance might not look like the most efficient approach, but it really doesn't take long[...]

Now, still in Proxmox, log on to each VM console using the user/pass you've configured and run:

```bash
sudo apt update && sudo apt upgrade -y
```

Then, when they finish, run:

```bash
sudo reboot
```

## SSH and tmux

Now SSH to each VM (this can be from another device on the home network, and it can also run tmux).

On macOS, run:

```bash
brew install tmux
```

Useful tmux commands:

- `^b + %` — split screen vertically (press `Ctrl+B`, release it, then `Shift+5`)
- `^b + "` — split screen horizontally
- `^b + arrows` — switch between panes on the current window
- `^b + x` — close the current session (a warning will ask if you are sure)
- `^d` — close the current tmux pane without warning

You can run `tmux`, then `^b + "` to split the screen horizontally, then do it again so you have 3 sections. SSH to worker 2 in the bottom window, then run `^b + up arrow` to move to the window a[...]

Now `^b + :` will open command mode, where you can type `setw synchronize-panes` to run the same command in multiple panes.

Now run:

```bash
sudo apt install qemu-guest-agent -y
```

Each command should now run for all VMs at the same time.

## Disable swap

We need to disable the swap file or kubelet may behave unpredictably:

```bash
sudo swapoff -a
sudo nano /etc/fstab
```

Comment out the line containing `swap.image`.

Now run:

```bash
cat /etc/hostname
```

Also check:

```bash
cat /etc/hosts
```

## Kernel modules

We need to load specific kernel modules for the overlay filesystem and bridged traffic:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Verify the file:

```bash
cat /etc/modules-load.d/k8s.conf
```

When the Linux system boots, it reads all files in that directory and automatically loads the listed modules. This ensures the overlay (for containers) and `br_netfilter` (for bridge networking) [...]

Load these modules into memory immediately without rebooting:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Overlay filesystem is used by Docker images, which is necessary for Kubernetes because it manages Docker containers.
`br_netfilter` allows the Linux kernel to pass traffic flowing through a network bridge (Layer 2) to the iptables/netfilter stack (Layer 3) for processing.

## Sysctl configuration

These system parameters should persist across reboots. We also have to tell the kernel to use them for IPv4 and IPv6 traffic:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Check the result:

```bash
cat /etc/sysctl.d/k8s.conf
```

Apply the new sysctl parameters to the current session:

```bash
sudo sysctl --system
```

## Install container runtime

Install `containerd`, the container runtime for the Kubernetes cluster:

```bash
sudo apt install containerd -y
```

Check if the service is up and running:

```bash
systemctl status containerd
```

Create the default configuration for the containerd daemon:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Review the file:

```bash
cat /etc/containerd/config.toml
```

Check the specific line we need to change:

```bash
cat /etc/containerd/config.toml | grep SystemdCgroup
```

The value is currently `false`; we need it to be `true`:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

Verify the change:

```bash
cat /etc/containerd/config.toml | grep SystemdCgroup
```

Restart and enable the service:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

You should see that the containerd service is both active and enabled (enabled means it will auto-start after reboot).

## Install Kubernetes components

Install `curl`, `gpg`, and add the Kubernetes repository:

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg
```

Download the public signing key:

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

> Important: if you see errors like `Failed to fetch https://pkgs.k8s.io/... 403 Forbidden` or the output shows an IPv6 address like `2600:9000:...` on port `443`, the issue is usually IPv6 connectivity, not the package repository itself. The IP `2600:9000:...` is an IPv6 address, and when IPv6 is broken or blocked in your home network, the connection to the external service can fail. In that case, force IPv4 for APT:
>
> ```bash
> sudo apt -o Acquire::ForceIPv4=true update
> sudo apt -o Acquire::ForceIPv4=true install -y apt-transport-https ca-certificates curl gpg
> ```
>
> Then retry the Kubernetes repository setup with IPv4 forced:
>
> ```bash
> curl -fsSL --ipv4 https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
> echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
> sudo apt -o Acquire::ForceIPv4=true update
> ```
>
> If your network does not support IPv6, you can also disable IPv6 on the VM temporarily or permanently:
>
> ```bash
> sudo nano /etc/sysctl.d/99-disable-ipv6.conf
> ```
>
> Add:
>
> ```bash
> net.ipv6.conf.all.disable_ipv6 = 1
> net.ipv6.conf.default.disable_ipv6 = 1
> net.ipv6.conf.lo.disable_ipv6 = 1
> ```
>
> Then apply:
>
> ```bash
> sudo sysctl -p /etc/sysctl.d/99-disable-ipv6.conf
> ```

Install the Kubernetes components:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

`kubelet`, `kubeadm`, and `kubectl` each have very distinct roles:
- `kubelet` = worker
- `kubeadm` = installer
- `kubectl` = remote control

The `apt-mark hold` command locks the package versions so APT will not upgrade them automatically. This is important because Kubernetes components must usually stay aligned with the same version. If you run a normal `apt update && apt upgrade`, APT may upgrade one package but not the others, which can cause cluster incompatibilities or `kubeadm` errors.

Example:

```bash
# lock those versions
sudo apt-mark hold kubelet kubeadm kubectl

# later, if you want to allow upgrades again
sudo apt-mark unhold kubelet kubeadm kubectl

# then update them together deliberately
sudo apt update
sudo apt install kubelet kubeadm kubectl

# and lock them again for stability
sudo apt-mark hold kubelet kubeadm kubectl
```

A good rule is: install them once, keep the versions matched, and only upgrade them deliberately as a set.

> **Troubleshooting**: If you encounter errors fetching from `pkgs.k8s.io` (403 Forbidden, connection issues, or missing release file), see [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for alternative installation methods.

## What is `--pod-network-cidr` and why is it important?

`--pod-network-cidr` defines the IP address range that Kubernetes pods use to communicate with each other inside the cluster.

It is different from your home network (for example `192.168.1.0/24`), because Kubernetes creates a separate virtual pod network. This internal network must not overlap with the real network used by your hosts or your router.

Example:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

This tells Kubernetes to assign pod IPs from the `10.244.0.0/16` range, such as:

- `10.244.0.2`
- `10.244.1.10`
- `10.244.2.25`

That range is separate from your LAN, so it does not conflict with devices like your router or Proxmox host.

### Why Flannel uses `10.244.0.0/16`

Flannel is the Container Network Interface (CNI) plugin we will install. Flannel is responsible for creating pod-to-pod connectivity in the cluster.

Flannel expects the pod CIDR to match the network it is configured to use. In this guide, we use the default Flannel network:

```bash
--pod-network-cidr=10.244.0.0/16
```

This is the simplest and most common option for a home lab Kubernetes cluster.

In short:
- `--pod-network-cidr` = the pod network range
- Flannel = the software that creates and manages that pod network
- `10.244.0.0/16` = the default network used by Flannel

If you change the pod network range, you must make sure the CNI plugin uses the same range or things will break.

## What is Flannel and why do we need it?

Flannel is a Kubernetes networking plugin that assigns IP addresses to Pods and routes traffic between them.

Without Flannel, Pods would not be able to talk to one another across nodes. The cluster could be initialized, but networking would not work correctly.

Typical installation:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Flannel creates the pod overlay network so that pods on the master and worker nodes can communicate as if they were part of one large private network.

## How do I know whether the master node is complete?

After you run `kubeadm init`, the master node is considered initialized when the control-plane pods are running and the API server is responding.

Check the status with:

```bash
sudo systemctl status kubelet
kubectl get nodes
kubectl cluster-info
kubectl get pods -n kube-system
kubectl get pods -n kube-flannel
```

You want to see:

- `kubelet` is `active (running)`
- `kubectl get nodes` shows the master node in `Ready` state
- `kubectl cluster-info` shows the Kubernetes API server and CoreDNS
- `kubectl get pods -n kube-system` shows all control-plane pods in `Running` state
- `kubectl get pods -n kube-flannel` shows Flannel pods in `Running` state

Example result:

```bash
kubectl get nodes
```

```text
NAME          STATUS   ROLES           AGE   VERSION
k8s-master    Ready    control-plane   5m    v1.32.0
```

If `kubelet` is inactive, it usually means the cluster has not been initialized yet. In that case, run:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Then create the kube config:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

After the cluster is initialized, Flannel should be installed and network communication should begin.

## Initialize the cluster

### Quick command map: which commands run where?

Run the following commands on every VM before the cluster is initialized:

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
sudo apt install qemu-guest-agent -y
sudo swapoff -a
sudo nano /etc/fstab
sudo modprobe overlay
sudo modprobe br_netfilter
sudo sysctl --system
sudo apt install containerd -y
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Run the following commands only on the master node:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Run the following command only on each worker node after the master generates the join token:

```bash
sudo kubeadm join 192.168.1.191:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Important: `kubeadm init` is executed once, on the master node only. The workers do not run it. They join using the command that the master prints after initialization.

Optional: Reboot all VMs:

```bash
sudo reboot
```

Probably not necessary, but it is good practice once you install many new system components.

Once rebooted, close the current tmux session and open each one individually. You can use `Ctrl + D` to close the tmux session.

Now SSH to each VM separately and run the following command on the master node only:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Do not change this IP prefix unless you have a reason to; it has nothing to do with your static DHCP scope on the router. The only requirement is that it cannot be the same as the one configured [...]

It also needs to match the CNI component we install shortly. Since we are going to use Flannel, an extremely popular and simple CNI, it is easiest to leave it as `10.244.0.0/16`.

When we run this command, it generates a `join` command we must use on the worker nodes to add them to the cluster.

> Note: This join command is only valid for 24 hours. If you create another worker node later, refresh it with:

```bash
kubeadm token create --print-join-command
```

Before running the join commands on the worker nodes, configure `kubectl` on the master node:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Install the CNI

We need to install a Container Network Interface (CNI) so the cluster can create an overlay network for communication between pods. If CNI is configured incorrectly, you will see all your nodes i[...]

The most popular CNI choices are Flannel and Calico. We will use Flannel.

Install it on the master node only:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

After that, copy the join command generated on the master node and paste it into the worker nodes. It says to run it as root, so use `sudo`.

Example:

```bash
sudo kubeadm join 192.168.1.191:6443 --token vyg76p.pz5k6dkrkaopvjhi --discovery-token-ca-cert-hash sha256:fb914ae294538ea1d35e18fab62df421f936dd68069eb877aa3b8e49634321a3
```

Your actual token and hash will be different; use the ones generated in your environment.

Check the cluster:

```bash
kubectl get nodes
```

You will see the worker node ROLE is empty, but that is normal in modern Kubernetes. The "worker" role is assumed by default for any nodes that are not masters.

This means your cluster works! At this stage you can deploy services to the new Kubernetes cluster.

## MetalLB for LoadBalancer services

You could just play with what you already created, but you will quickly notice a big limitation: you do not have the Kubernetes service called `LoadBalancer` available.

You can deploy `NodePort` services or use `HostNetwork`, but if you want a more cloud-like solution, adding MetalLB is the easiest approach.

Update the `kube-proxy` config map:

```bash
kubectl edit configmap -n kube-system kube-proxy
```

In the `ipvs` section, change `strictARP` from `false` to `true`.

For `vim`, press `i` to enter insert mode, make the change, then press `Esc` and type `:wq` to save and quit.

Install MetalLB:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

Check the pods:

```bash
kubectl get pods -n metallb-system
```

You should see 4 pods running. It may take a few moments before they are all up.

### Configure the IP pool

Create a file named `proxmox-ip-pool.yaml` in your home directory and paste this:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: marek-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.245 # CHANGE THIS to your desired range
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: layer2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - marek-pool
```

Apply it:

```bash
kubectl apply -f proxmox-ip-pool.yaml
```

## Test LoadBalancer

Create another YAML file, for example `nginx-test.yaml`:

```yaml
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
```

Deploy it:

```bash
kubectl apply -f nginx-test.yaml
```

Then check the service:

```bash
kubectl get svc nginx-service
```

You should see the cluster IP and external IP. The external IP is the one assigned by MetalLB.

Open it in your browser, e.g.:

```text
http://192.168.1.240:80
```

You should see the default Nginx page: "Welcome to nginx".

## Final thoughts

You can now build any services you like, add an ingress controller, and do whatever you want, because your Kubernetes cluster is now fully functional.
