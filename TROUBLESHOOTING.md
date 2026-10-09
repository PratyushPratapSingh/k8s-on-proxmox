# Troubleshooting Guide

## Issue: "Failed to fetch https://pkgs.k8s.io/core:/stable:/v1.32/deb/" (403 Forbidden or Cannot reach)

The Kubernetes package repository at `pkgs.k8s.io` experiences intermittent availability issues. Here are several solutions:

### Solution 1: Retry with Network Troubleshooting (Quick Fix)

First, verify your network connectivity:

```bash
# Test basic internet connectivity
ping 8.8.8.8

# Test DNS resolution
nslookup pkgs.k8s.io

# Test direct curl access with verbose output
curl -v https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release
```

If connectivity is fine, retry the installation with timeouts and retries:

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings

# Try downloading the key with retries and timeout
curl -fsSL --retry 5 --retry-delay 2 --connect-timeout 10 \
  https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update and install
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Solution 2: Use Ubuntu's Official Kubernetes Packages (Recommended for Home Labs)

Ubuntu includes Kubernetes packages in its official repositories. This is the most stable approach for home-lab environments:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

This installs versions maintained by Ubuntu. Check what version is available:

```bash
apt search kubeadm | grep "^kubeadm/"
```

### Solution 3: Install from Snap (Alternative)

If APT repositories fail entirely, you can use Snap:

```bash
sudo snap install kubectl --classic
sudo snap install kubeadm --classic
sudo snap install kubelet --classic
```

### Solution 4: Downgrade Kubernetes Version

If you're experiencing persistent issues with v1.32, try a slightly older version by specifying the version:

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings

# Try v1.31 instead
curl -fsSL --retry 5 https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Solution 5: Manual Binary Installation

Download Kubernetes binaries directly:

```bash
K8S_VERSION="v1.32.0"  # Change as needed

# Create directory
mkdir -p ~/k8s-bins
cd ~/k8s-bins

# Download the binaries
curl -L https://dl.k8s.io/${K8S_VERSION}/bin/linux/amd64/kubectl -o kubectl
curl -L https://dl.k8s.io/${K8S_VERSION}/bin/linux/amd64/kubelet -o kubelet
curl -L https://dl.k8s.io/${K8S_VERSION}/bin/linux/amd64/kubeadm -o kubeadm

# Make executable and move to PATH
chmod +x kubectl kubelet kubeadm
sudo mv kubectl kubelet kubeadm /usr/local/bin/
```

## Network/Proxy Issues

If you're behind a corporate proxy or firewall:

```bash
# Test with explicit proxy (adjust proxy URL/port as needed)
curl -fsSL -x http://proxy.example.com:8080 https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key

# Configure apt to use proxy permanently
sudo nano /etc/apt/apt.conf.d/proxy.conf
# Add: Acquire::http::Proxy "http://proxy.example.com:8080/";
# Add: Acquire::https::Proxy "http://proxy.example.com:8080/";
```

## Check Repository Status

Monitor the repository status at: https://groups.google.com/forum/#!forum/kubernetes-security-announce

Or try these diagnostic commands:

```bash
# Check if DNS works
getent hosts pkgs.k8s.io

# Check system time (wrong time can cause SSL certificate issues)
date

# Check for APT errors
sudo apt clean
sudo rm -rf /var/lib/apt/lists/*
sudo mkdir -p /var/lib/apt/lists/partial
sudo apt update

# View APT sources
cat /etc/apt/sources.list.d/kubernetes.list
```

## Summary of Solutions (In Order of Preference)

1. **Use Ubuntu's official repository** - Most stable for home labs, no version-specific URLs
2. **Retry with network troubleshooting** - Might be a temporary issue with pkgs.k8s.io
3. **Use Snap packages** - When APT completely fails
4. **Downgrade to v1.31** - If specific v1.32 version is problematic
5. **Download binaries directly** - Last resort, manual installation
6. **Check network/proxy** - If behind corporate firewall

Choose the solution that best fits your network environment and version requirements.
