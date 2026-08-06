---
title: Run Podman on macOS with Lima
url: https://devopstales.github.io/virtualization/lima-podman-macos/
date: 2026-03-04
keywords: Lima, Podman, Linux on macOS, Container Runtime, Docker alternative
---


Running Podman on macOS through Lima provides a lightweight, Docker-compatible container runtime without the overhead of Docker Desktop. This setup is ideal for developers who want a rootless, daemonless container experience on Mac with minimal resource consumption.

<!--more-->

![Podman on Lima macOS](/img/include/lima-podman-macos.webp)

## What is Podman?

Podman (Pod Manager) is a daemonless, rootless container engine for developing, managing, and running OCI containers. Unlike Docker, Podman doesn't require a running daemon, making it more secure and lightweight. It's fully compatible with Docker CLI commands and supports Kubernetes-native workflows.

## Why Podman on Lima?

Running Podman inside a Lima VM on macOS offers several advantages:

- **Rootless by default**: No need for elevated privileges
- **Daemonless architecture**: Containers run as regular processes
- **Docker-compatible**: Use familiar `docker` commands with `podman`
- **Lightweight**: Less resource overhead than Docker Desktop
- **Kubernetes-native**: Built-in support for pods and Kubernetes YAML

## Installation

### Install Lima

```bash
# Using Homebrew
brew install lima
```

### Install Podman

```bash
# Install Podman CLI on macOS (for client-side tools)
brew install podman
```

## Quick Setup with Colima

The easiest way to run Podman with Lima is through Colima, which supports Podman as a runtime.

### Start Colima with Podman

```bash
# Start Colima with Podman runtime
colima start --runtime podman
```

This creates a Lima VM with Podman pre-configured and ready to use.

### Verify Installation

```bash
# Check Podman version
podman --version

# List containers
podman ps

# Run a test container
podman run hello-world
```

## Manual Lima Configuration

For more control, create a custom Lima VM configuration with Podman.

### Create Custom Lima Instance

```bash
# Create a new Lima instance configuration
limactl create podman
```

This opens an editor with the default configuration. Modify it to include Podman setup:

```yaml
# Lima configuration for Podman
images:
  - location: "https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64.img"
    arch: "x86_64"

cpus: 4
memory: "4GiB"
disk: "50GiB"

provision:
  - mode: system
    script: |
      #!/bin/bash
      # Install Podman
      apt-get update
      apt-get install -y podman podman-docker

  - mode: user
    script: |
      #!/bin/bash
      # Configure Podman socket for Docker compatibility
      mkdir -p ~/.docker
      echo '{"credHelpers": {}, "credsStore": "desktop"}' > ~/.docker/config.json

portForwards:
  - guestPort: 8080
    hostPort: 8080

mounts:
  - location: "~/Projects"
    mountPoint: "/home/ubuntu/Projects"
    writable: true
```

### Start the Instance

```bash
# Start the Lima VM
limactl start podman

# SSH into the VM
lima podman ssh

# Or run commands directly
lima podman run hello-world
```

## Docker Compatibility

Podman provides Docker CLI compatibility out of the box. You can use Docker commands directly:

```bash
# Create an alias for Docker compatibility
alias docker=podman

# Or use podman-docker wrapper
podman-docker run -d -p 8080:80 nginx
```

### Configure Docker Socket

For tools that expect the Docker socket:

```bash
# Inside the Lima VM
sudo systemctl enable --now podman.socket
sudo ln -s /run/podman/podman.sock /var/run/docker.sock
```

## Common Podman Commands

```bash
# List running containers
podman ps

# List all containers (including stopped)
podman ps -a

# List images
podman images

# Run a container
podman run -d -p 8080:80 nginx

# Build an image
podman build -t myapp .

# Create a pod
podman pod create -n mypod

# Run container in a pod
podman run --pod mypod -d nginx

# Generate Kubernetes YAML
podman generate kube mypod > pod.yaml

# Play Kubernetes YAML
podman play kube pod.yaml
```

## Podman Machine (Alternative)

Podman also offers its own machine management:

```bash
# Initialize Podman machine
podman machine init

# Start Podman machine
podman machine start

# List machines
podman machine list

# SSH into machine
podman machine ssh
```

## Volume Mounts and File Sharing

Lima automatically shares files between macOS and the VM. Configure mounts in your Lima config:

```yaml
mounts:
  - location: "~/Projects"
    mountPoint: "/home/ubuntu/Projects"
    writable: true
  - location: "/tmp"
    mountPoint: "/tmp"
    writable: true
```

Then use volumes in Podman:

```bash
podman run -v ~/Projects:/app -d myapp
```

## Networking

### Port Forwarding

Lima handles port forwarding automatically. Configure in your Lima config:

```yaml
portForwards:
  - guestPort: 8080
    hostPort: 8080
  - guestPortRange: [3000, 3010]
    hostPortRange: [3000, 3010]
```

### Access Containers from Host

Containers running in the Lima VM are accessible from macOS through the forwarded ports:

```bash
# Run container with port mapping
podman run -d -p 8080:80 nginx

# Access from macOS
curl http://localhost:8080
```

## Kubernetes Integration

Podman has native Kubernetes support:

```bash
# Create a pod with multiple containers
podman pod create -n webapp
podman run --pod webapp -d nginx
podman run --pod webapp -d redis

# Generate Kubernetes manifest
podman generate kube webapp > webapp.yaml

# Deploy to Kubernetes cluster
kubectl apply -f webapp.yaml
```

## Troubleshooting

### Check Lima VM Status

```bash
limactl list
lima status podman
```

### View Logs

```bash
limactl logs podman
```

### Restart Podman Service

```bash
# Inside the Lima VM
sudo systemctl restart podman.socket
```

### Reset Everything

```bash
# Delete Lima instance
limactl delete podman

# Recreate
limactl create podman
limactl start podman
```

### Connection Issues

If Podman commands fail to connect:

```bash
# Check socket exists
ls -la /run/podman/podman.sock

# Verify socket is active
systemctl status podman.socket

# Check permissions
ls -la /var/run/docker.sock
```

## Benefits Over Docker Desktop

| Feature | Podman on Lima | Docker Desktop |
|---------|---------------|----------------|
| **Daemon** | Daemonless | Requires daemon |
| **Root Access** | Rootless by default | Requires elevated privileges |
| **Resource Usage** | Lightweight | Higher overhead |
| **Kubernetes** | Native pod support | Requires K8s enablement |
| **License** | Apache 2.0 | Proprietary (free tier limited) |
| **Pods** | First-class citizen | Limited support |

## Common Use Cases

### Development Environment

```bash
# Start database and app in a pod
podman pod create -n dev
podman run --pod dev -d postgres:15
podman run --pod dev -d redis:7
podman run --pod dev -p 3000:3000 -d myapp
```

### CI/CD Testing

```bash
# Build and test in isolated environment
podman build -t test-image .
podman run --rm test-image npm test
```

### Kubernetes Development

```bash
# Develop locally with Kubernetes semantics
podman play kube deployment.yaml
podman generate kube mypod > local-test.yaml
```

## Conclusion

Running Podman on macOS through Lima provides a lightweight, secure alternative to Docker Desktop. With rootless containers, daemonless architecture, and native Kubernetes support, this setup is ideal for developers who want full container capabilities without the overhead. Whether you're developing microservices, testing Kubernetes manifests, or running CI/CD pipelines, Podman on Lima delivers a production-like container experience on macOS.

The combination of Lima's efficient virtualization and Podman's Kubernetes-native approach makes this stack particularly well-suited for cloud-native development workflows on Mac hardware.

