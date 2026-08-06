---
title: Speed up docker pull with lazypull
url: https://devopstales.github.io/kubernetes/lazyimage/
date: 2021-08-02
keywords: Kubernetes, k8s, kubernetes deployment, containerd, lazypull, stargz
---



In this post I will show you the solutions to speed up the container downloads.
<!--more-->

According to Googles's analisys "pulling packages accounts for 76% of container start time, but only 6.4% of that data is read" This of course affecting various kinds of workloads on our Kubernetes clusters like cold-start of containers, serverless functions, build and CI/CD. In the community, workarounds are known but they still have unavoidable drawbacks. Let's check this workarounds:

### Background
When you run a container the command will pull images from a repository if they are not already available locally. The image is downloaded but not in one big file then multiple smaller one. The runtime engin downloads this files simultaneously to speed up this process. In the background all part is a tar-file downloaded with wget the extracted.

The AUFS storage driver is a common default in container runtime. The AUFS driver takes advantage of the AUFS file system’s layering and copy-on-write (COW) capabilities while also accessing the file system underlying AUFS directly. The driver creates a new directory in the underlying file system for each layer it stores. As a union file system it does not store data directly on disk, but instead uses another file system (e.g. ext4) as underlying storage. A union mount point provides a view of multiple directories in the underlying file system.

Using a HelloBench tool that they wrote, the authors analyse 57 different container images pulled from the Docker Hub. Across these images there are 550 nodes and and 19 roots. They realized that the average uncompressed image is 15 x larger that the amount of image data needed for container startup. 

For solving this problem several solution have been proposed including [CernVM-FS](https://cernvm.cern.ch/fs/), [Microsoft Teleportation](https://stevelasker.blog/2019/10/29/azure-container-registry-teleportation/), [Google CRFS](https://github.com/google/crfs) and [Dragonfly](https://d7y.io/en-us/)

### Standard compatible solution

Containerd started [Stargz Snapshotter](https://github.com/containerd/stargz-snapshotter) as a plugin to improve the pull performance. It enables to lazy pull container images leveraging [stargz image format from Google](https://github.com/google/crfs). 

![lazypull](/img/include/lazypull1.webp)

The lazy pull here means containerd doesn’t download the entire image on pull operation but fetches necessary contents on-demand. This shortens the container startup latency from tens of seconds into a few seconds at the best.

![Benchmarking result from the project repository.](/img/include/lazypull2.webp)

### Quick start
Containerd supports lazy pulling since version 1.4. Stargz Snapshotter is the plugin that enables containerd to handle eStargz.

```bash
nano /etc/containerd/config.toml
...
[proxy_plugins]
  [proxy_plugins.stargz]
    type = "snapshot"
    address = "/run/containerd-stargz-grpc/containerd-stargz-grpc.sock"

# Use stargz snapshotter through CRI
[plugins."io.containerd.grpc.v1.cri".containerd]
  snapshotter = "stargz"
```

```bash
nerdctl --snapshotter=stargz run -it --rm docker.io/stargz-containers/fedora:30-esgz
```

CRI-O experimentally supports lazy pulling. The plugin that enables this is called [Additional Layer Store](https://github.com/containerd/stargz-snapshotter/blob/main/docs/INSTALL.md#whats-stargz-snapshotter-and-stargz-store).

```bash
nano /etc/containers/storage.conf
# Additional Layer Store is supported only by overlay driver as of now 
[storage]
driver = "overlay"
graphroot = "/var/lib/containers/storage"
runroot = "/run/containers/storage"

# Interact with Additional Layer Store over a directory
[storage.options]
additionallayerstores = ["/path/to/additional/layer/store:ref"]
```

![eStargz in container workflow](/img/include/lazypull3.webp)

### Build images
You can create an eStarge image using eStargz-aware image builders (e.g. [Kaniko](https://github.com/GoogleContainerTools/kaniko)) or image converters (e.g. ctr-remote and nerdctl).

[ctr-remote](https://github.com/containerd/stargz-snapshotter/blob/main/docs/ctr-remote.md) is a CLI for converting an OCI/Docker image into eStargz.

```bash
ctr-remote image pull docker.io/library/ubuntu:21.04
ctr-remote image optimize --oci \
    docker.io/library/ubuntu:21.04 docker.io/devopstales/ubuntu:21.04-esgz
ctr-remote image push docker.io/devopstales/ubuntu:21.04-esgz
```

nerdctl supports creating eStargz images

```bash
nerdctl build -t docker.io/devopstales/foo:1 .
nerdctl image convert --estargz --oci \
    docker.io/devopstales/foo:1 docker.io/devopstales/foo:1-esgz
nerdctl push docker.io/devopstales/foo:1-esgz
```

[Kaniko](https://github.com/GoogleContainerTools/kaniko) is an image builder runnable in containers and Kubernetes. Since v1.5.0, it experimentally supports building eStargz. 7

```bash
docker run --rm -e GGCR_EXPERIMENT_ESTARGZ=1 \
    -v /tmp/context:/workspace \
    -v ~/.docker/config.json:/kaniko/.docker/config.json:ro \
    gcr.io/kaniko-project/executor:v1.6.0 \
      --destination "docker.io/devopstales/sample:esgz"
```

