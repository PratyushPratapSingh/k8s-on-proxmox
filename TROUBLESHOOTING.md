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

### Solution 2: Use an Alternative Repository (Recommended)

If `pkgs.k8s.io` continues to fail, use the Google Cloud Debian repository instead:

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg

# Add Google Cloud public key
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys BA07F4FB

# Add the Google Cloud Kubernetes repository
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update and install
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Solution 3: Downgrade Kubernetes Version

If you're experiencing persistent issues with v1.32, try a slightly older version:

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

### Solution 4: Manual Installation with Specific Package Versions

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg

# Install from Ubuntu's official repository (usually slightly older versions)
sudo apt install -y kubelet kubeadm kubectl

# Or specify exact versions if available
sudo apt install -y kubelet=1.32.* kubeadm=1.32.* kubectl=1.32.*

sudo apt-mark hold kubelet kubeadm kubectl
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

1. **Retry with network troubleshooting** - Might be a temporary issue
2. **Use Google Cloud repository** - Most reliable alternative
3. **Downgrade to v1.31** - If specific version is problematic
4. **Use Ubuntu's repository** - Slightly older but very stable
5. **Check network/proxy** - If behind corporate firewall

Choose the solution that best fits your network environment and version requirements.
