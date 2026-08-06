---
title: Free Docker Desktop Alternative For Mac And Windows
url: https://devopstales.github.io/kubernetes/docker-desktop-alternatives/
date: 2021-09-20
keywords: docker, Alternative, podman, Lima, Linux virtual machines, on macOS, Rancher Desktop, mikrok8s, minikube, macos, windows, choco, Hyper-V, Containderd is the new Docker!!
---



At Aug. 31, 2022 Docker announced a new subscription plan for Docker Desktop. So we will Check the best alternatives for docker desktop on Windows an MacOS.

<!--more-->

Docker Desktop remain free for:

* Small businesses with fewer than 250 employees and less than $10 million in annual revenue.
* Personal use.
* Educational institutions.
* Non-commercial open-source projects.

For professional usage, Docker Desktop will require a paid subscription (either Pro, Team or Business), which starts at $5 per month and goes up from there.

## podman
Podman is a tool for running OCI containers and pods.

### MacOS
You can do this from a MacOS desktop as long as you have access to a linux box either running inside of a VM on the host, or available via the network. Podman includes a command, podman machine that automatically manages VM’s.

```bash
brew install podman

podman machine init
podman machine start

podman info

podman run -it --rm -d -p 8080:80 --name web nginx
```

### Windows
Podman can be run in the Windows Subsystem for Linux system.

```bash
wsl.exe --install

bash

sudo apt-get -y update
sudo apt-get -y install podman

podman info

podman run -it --rm -d -p 8080:80 --name web nginx
```

## lima and Colima
Lima (Linux virtual machines, on macOS) launches Linux virtual machines with automatic file sharing, port forwarding, and containerd.

### MacOS

```bash
brew install lima
limactl start
lima nerdctl run -it --rm alpine
```

> NOTE: ARM Mac requires installing a patched version of QEMU, see [Lima documentation](https://github.com/lima-vm/lima#does-lima-work-on-arm-mac)

If you want to use Kubernetes on Lima there is a project called Colima.

```bash
brew install lima docker kubectl
curl -LO https://raw.githubusercontent.com/abiosoft/colima/v0.1.10/colima && sudo install colima /usr/local/bin/colima

colima version

colima start --with-kubernetes
# configure vm
colima stop
colima start --cpu 4 --memory 8
```

Lima is already adopted by Rancher Desktop to run k3s on macOS.

## Rancher Desktop
Rancher Desktop is an open-source project to bring Kubernetes and container management to the desktop. Windows and macOS versions of Rancher Desktop are available for download.

### MacOS
Rancher Desktop requires the following on macOS.

* macOS 10.10 or higher.
* Intel CPU with VT-x.
* Persistent internet connection.

Apple Silion (M1) support is planned, but not currently implemented.

### Windows
Rancher Desktop requires the following on Windows:

* Windows 10, at least version 1903.
* Running on a machine with virtualization capabilities.
* Persistent internet connection.

Rancher Desktop requires Windows Subsystem for Linux on Windows; this will automatically be installed as part of the Rancher Desktop setup. Manually downloading a distribution is not necessary.

### Install
Download and install the newes version fro [GitHub](https://github.com/rancher-sandbox/rancher-desktop/releases) Then Start it.

![Rancher Desktop Architecture](/img/include/racherdesktop01.webp)

Rancher Desktop installs a new Linux VM in WSL2 that has a Kubernetes cluster based on k3s as well as installs various components in it such as KIM (for building docker images on the cluster), helm cli and the Traefik Ingress Controller

![Rancher Desktop GUI1](/img/include/racherdesktop02.webp)

![Rancher Desktop GUI2](/img/include/racherdesktop03.webp)

## mikrok8s
MicroK8s is the simplest production-grade upstream K8s distribution.

### MacOS
To run mikrok8s on macOS you need at least 8GB of RAM:

```bash
brew install ubuntu/microk8s/microk8s
microk8s install

microk8s status --wait-ready
microk8s kubectl get nodes
microk8s kubectl get services

# enable addons
microk8s enable dns storage

# manage vm
microk8s stop
microk8s start
```

### Windows
To run mikrok8s on windows the requirements are:

* A Windows 10 machine with at least 8 GB of RAM and 40 GB storage
* If you have Windows 10 Home edition, you will also need to install VirtualBox (Windows 10 Professional, Enterprise and Student editions include Hyper-v for virtualisation).

MicroK8s has a Windows installer that will take care of setting up the software for you. [Download the latest installer here.](https://github.com/ubuntu/microk8s/releases)

![mikrok8s installer](/img/include/mikrok8s01.webp)

The installer checks if Hyper-V is available and switched on. If you don’t have Hyper-v (e.g. on Windows 10 Home edition) it is possible to use VirtualBox as an alternative.

You can now configure MicroK8s - the minimum recommendations are already filled in.

![mikrok8s Config](/img/include/mikrok8s02.webp)

Now you can tart an use micro k8s:

```bash
microk8s status --wait-ready
microk8s kubectl get nodes
microk8s kubectl get services

# enable addons
microk8s enable dns storage

# manage vm
microk8s stop
microk8s start
```

## minikube
minikube implements a local Kubernetes cluster on macOS, Linux, and Windows. minikube's primary goals are to be the best tool for local Kubernetes application development and to support all Kubernetes features that fit.

### MacOS

```bash
brew install hyperkit minikube docker kubectl

# configure vm
minikube config set memory 16384

# set driver
minikube start --driver hyperkit
minikube config set driver hyperkit

# configure docker cli to connect minikube vm
eval $(minikube -p minikube docker-env)
```

### Windows

Install with exe from powershell:
```bash
curl -Lo minikube.exe https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe
New-Item -Path "c:\" -Name "minikube" -ItemType "directory" -Force
Move-Item .\minikube.exe c:\minikube\minikube.exe -Force

$oldpath=[Environment]::GetEnvironmentVariable("Path", [EnvironmentVariableTarget]::Machine)
if($oldpath -notlike "*;C:\minikube*"){`
  [Environment]::SetEnvironmentVariable("Path", $oldpath+";C:\minikube", [EnvironmentVariableTarget]::Machine)`
}
```

Install wit Windows Package Manager:
```bash
winget install minikube
```

using Chocolatey:
```bash
choco install minikube
```

> VirtualBox and VMware Workstation (and VMware Player) are "level 2 hypervisors." Hyper-V and VMware ESXi are "level 1 hypervisors."
> So wen Hyper-V is enabled your Windows 10 pc become a Virtualization Host. Hyper-V blocks all other Hyper Visors like VirtualBox from calling VT hardware.

```bash
# to use with Hyper-V yo need to install
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All

# configure vm
minikube config set memory 16384

# set driver
minikube start --driver hyperv
minikube config set driver hyperv
```

