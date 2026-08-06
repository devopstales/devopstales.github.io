---
title: Containers vs. Pods - Deepdyve
url: https://devopstales.github.io/kubernetes/containers-vs-pods/
date: 2022-04-29
keywords: kubernetes deployment, Docker, Podman, Pod, cgroup, namespace
---


In this post we will take a look at the difference between containers and pods.

<!--more-->

With the wide usage of Docker/OICD Containers, they become a replacement for vm-s. This solution is based on Micro services best prentices that means  you are using one service per container.
This can be a problem some situation and blocks moving from vm to container. While thar is a few workaround for running multiple service, Kubernetes hes a solution in a for of 'Pods'. Pods are the smallest deployable units of Kubernetes. It is a group of one or more containers, with shared resources.
Every pod gets a unique IP. More from this in the [networking post](/kubernetes/kubernetes-networking-1/). This means you cannot run the same ports on different container in the same namespace. Every container in a pod gets an isolated filesystem and that from inside one container, you don't see processes running in other containers of the same pod. But containers in one pod can communicate via shared memory! 

Containers dose not necessary means Docker. There are more container technologies lik LXC, OpenVZ, and more. The difference in this technologies are the different type of isolation for processes running in the container. As we talked about [in a prewious post](/kubernetes/kubernetes-deprecated-docker-containderd-docker/) Docker/OICD Containers standards are based on the OCI Runtime Spec. 


### Namespace Isolation

Let's see what isolation primitives were created when I started the container:

```bash
# Look up the container in the process tree.
$ ps auxf
USER       PID  ...  COMMAND
...
root      4707       /usr/bin/containerd-shim-runc-v2 -namespace moby -id cc9466b3e...
root      4727        \_ nginx: master process nginx -g daemon off;
systemd+  4781            \_ nginx: worker process
systemd+  4782            \_ nginx: worker process

# Find the namespaces used by 4727 process.
$ sudo lsns
        NS TYPE   NPROCS   PID USER    COMMAND
...
4026532157 mnt         3  4727 root    nginx: master process nginx -g daemon off;
4026532158 uts         3  4727 root    nginx: master process nginx -g daemon off;
4026532159 ipc         3  4727 root    nginx: master process nginx -g daemon off;
4026532160 pid         3  4727 root    nginx: master process nginx -g daemon off;
4026532162 net         3  4727 root    nginx: master process nginx -g daemon off;
```

As you can see Docker user multiple namespaces to isolate the conatiners:

* mnt - isolated mount table 
* uts - the container has its own hostname and domain name
* ipc, pid - only to processes inside the same container can communicate with eache other
* net - the container gets its own network stack

User ID namespace is not used by default but you can run docker with [rootless](https://docs.docker.com/engine/security/rootless/) and use [user isolation](https://docs.docker.com/engine/security/userns-remap/).

There is one other major namespace and that is `Cgroup`. Cgroup can be use to limits hungry processes to accidentally consume all the host's resources.

### Check Cgroup limits

We can examining he corresponding subtree in the cgroup virtual filesystem. `cgroupfs` is mounted as `/sys/fs/cgroup` and for processes `/proc/<PID>/cgroup`.

Lets run a container for testing:

```bash
docker run --name foo --rm -d --memory='512MB' --cpus='0.5' nginx
```

```bash
# et pid for container
PID=$(docker inspect --format '{{.State.Pid}}' foo)

# Check cgroupfs node for the container main process (4727).
$ cat /proc/${PID}/cgroup
11:freezer:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
10:blkio:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
9:rdma:/
8:pids:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
7:devices:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
6:cpuset:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
5:cpu,cpuacct:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
4:memory:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
3:net_cls,net_prio:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
2:perf_event:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
1:name=systemd:/docker/cc9466b3eb67ca374c925794776aad2fd45a34343ab66097a44594b35183dba0
0::/system.slice/containerd.service

# Check the memory limit.
$ cat /sys/fs/cgroup/memory/docker/${ID}/memory.limit_in_bytes
536870912  # Yay! It's the 512MB we requested!

# See the CPU limits.
ls /sys/fs/cgroup/cpu/docker/${ID}
```

### Examining a Kubernetes pod

To keep the Docker Containers and Kubernetes Pods fair comparison I use a Docker as the engine in Kubernetes. Now I start a pod for testing:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
    - name: app
      image: docker.io/kennethreitz/httpbin
      ports:
        - containerPort: 80
      resources:
        limits:
          memory: "256Mi"
    - name: sidecar
      image: curlimages/curl
      command: ["/bin/sleep", "3650d"]
      resources:
        limits:
          memory: "128Mi"
EOF
```

The actual pod inspection should be done on the Kubernetes cluster node:

```bash
$ ps auxf
USER       PID  ...  COMMAND
...
root      4947         \_ containerd-shim -namespace k8s.io -workdir /mnt/sda1/var/lib/containerd/...
root      4966             \_ /pause
root      4981         \_ containerd-shim -namespace k8s.io -workdir /mnt/sda1/var/lib/containerd/...
root      5001             \_ /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
root      5016                 \_ /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
root      5018         \_ containerd-shim -namespace k8s.io -workdir /mnt/sda1/var/lib/containerd/...
100       5035             \_ /bin/sleep 3650d
```

The above three process groups were created during the pod startup. That's interesting because in the manifest, only two containers, `httpbin` and `sleep`, were requested.

Every Kubernetes Pod includes an empty pause container, which bootstraps the Pod to establish all of the cgroups, reservations, and namespaces before its individual containers are created. The pause container image is always present, so the pod resource allocation happens instantaneously as containers are created.

Display pause containers:

```bash
docker ps | grep -I pause

80282e0baa43   mirantis/ucp-pause:3.4.9   "/pause"   3 minutes ago   Up 3 minutes   k8s_POD_foo-wxk5l_default_030fc501-ec75-4675-a742-d19929818065_0
```

Here is how the namespaces look like on the cluster node:

```bash
sudo lsns
        NS TYPE   NPROCS   PID USER            COMMAND
4026532614 net         4  4966 root            /pause
4026532715 mnt         1  4966 root            /pause
4026532716 uts         4  4966 root            /pause
4026532717 ipc         4  4966 root            /pause
4026532718 pid         1  4966 root            /pause
4026532719 mnt         2  5001 root            /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
4026532720 pid         2  5001 root            /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
4026532721 mnt         1  5035 100             /bin/sleep 3650d
4026532722 pid         1  5035 100             /bin/sleep 3650d
```

So, much like the Docker container in the first section, the pause container gets all five namespaces - net, mnt, uts, ipc, and pid. But apparently, httpbin and sleep containers get just by two namespaces mnt and pid.

With the proc we can get tha semo for all the pods:

```bash
# httpbin container
sudo ls -l /proc/5001/ns
...
lrwxrwxrwx 1 root root 0 Apr 24 14:05 ipc -> 'ipc:[4026532717]'
lrwxrwxrwx 1 root root 0 Apr 24 14:05 mnt -> 'mnt:[4026532719]'
lrwxrwxrwx 1 root root 0 Apr 24 14:05 net -> 'net:[4026532614]'
lrwxrwxrwx 1 root root 0 Apr 24 14:05 pid -> 'pid:[4026532720]'
lrwxrwxrwx 1 root root 0 Apr 24 14:05 uts -> 'uts:[4026532716]'

# sleep container
sudo ls -l /proc/5035/ns
...
lrwxrwxrwx 1 100 101 0 Apr 24 14:05 ipc -> 'ipc:[4026532717]'
lrwxrwxrwx 1 100 101 0 Apr 24 14:05 mnt -> 'mnt:[4026532721]'
lrwxrwxrwx 1 100 101 0 Apr 24 14:05 net -> 'net:[4026532614]'
lrwxrwxrwx 1 100 101 0 Apr 24 14:05 pid -> 'pid:[4026532722]'
lrwxrwxrwx 1 100 101 0 Apr 24 14:05 uts -> 'uts:[4026532716]'
```

While it might be tricky to notice, the httpbin and sleep containers actually reuse the net, uts, and ipc namespaces of the pause container!

With kubernetes config the `hostIPC`, `hostNetwork`, and `hostPID` flags can make the containers use the corresponding host's namespaces.

### Inspecting pod's cgroups

Check the cgroups for pods:

```bash
$ sudo systemd-cgls
Control group /:
-.slice
├─kubepods
│ ├─burstable
│ │ ├─pod4a8d5c3e-3821-4727-9d20-965febbccfbb
│ │ │ ├─f0e87a93304666766ab139d52f10ff2b8d4a1e6060fc18f74f28e2cb000da8b2
│ │ │ │ └─4966 /pause
│ │ │ ├─dfb1cd29ab750064ae89613cb28963353c3360c2df913995af582aebcc4e85d8
│ │ │ │ ├─5001 /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
│ │ │ │ └─5016 /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
│ │ │ └─097d4fe8a7002d69d6c78899dcf6731d313ce8067ae3f736f252f387582e55ad
│ │ │   └─5035 /bin/sleep 3650d
```

So, the pod itself gets a parent node, and every container can be tweaked separately as well.

### Implementing Pods with Docker

Because Pod under the hood is implemented as a bunch of shared namespaced containers with a common cgroup parent, I will try to reproduce this with Docker because Docker cannot manage pods. 

Docker allows creating a container that reuses an existing network namespace. I'll use an extra package to simplify dealing with cgroups:

```bash
sudo apt-get install cgroup-tools
```

Firstly, a parent cgroup entry needs to be configured. For the sake of simplecity, I'll use only cpu and memory controllers:

```bash
sudo cgcreate -g cpu,memory:/pod-foo

# Check if the corresponding folders were created:
ls -l /sys/fs/cgroup/cpu/pod-foo/
ls -l /sys/fs/cgroup/memory/pod-foo/
```

Secondly, a sandbox (paus) container should be created:

```bash
$ docker run -d --rm \
  --name foo_sandbox \
  --cgroup-parent /pod-foo \
  --ipc 'shareable' \
  alpine sleep infinity
```

Lastly, starting the actual containers reusing the namespaces of the sandbox container:

```bash
# app (httpbin)
$ docker run -d --rm \
  --name app \
  --cgroup-parent /pod-foo \
  --network container:foo_sandbox \
  --ipc container:foo_sandbox \
  kennethreitz/httpbin

# sidecar (sleep)
$ docker run -d --rm \
  --name sidecar \
  --cgroup-parent /pod-foo \
  --network container:foo_sandbox \
  --ipc container:foo_sandbox \
  curlimages/curl sleep 365d
```

I couldn't share the uts namespace between containers, because Docker not allow to configure that. You can use only just the host's uts namespace. But apart from the uts namespace, it's a success!

The cgroups look much like the ones created by Kubernetes itself:

```bash
$ sudo systemd-cgls memory
Controller memory; Control group /:
├─pod-foo
│ ├─488d76cade5422b57ab59116f422d8483d435a8449ceda0c9a1888ea774acac7
│ │ ├─27865 /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
│ │ └─27880 /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
│ ├─9166a87f9a96a954b10ec012104366da9f1f6680387ef423ee197c61d37f39d7
│ │ └─27977 sleep 365d
│ └─c7b0ec46b16b52c5e1c447b77d67d44d16d78f9a3f93eaeb3a86aa95e08e28b6
│   └─27743 sleep infinity
```

The global list of namespaces also looks familiar:

```bash
$ sudo lsns
        NS TYPE   NPROCS   PID USER    COMMAND
...
4026532157 mnt         1 27743 root    sleep infinity
4026532158 uts         1 27743 root    sleep infinity
4026532159 ipc         4 27743 root    sleep infinity
4026532160 pid         1 27743 root    sleep infinity
4026532162 net         4 27743 root    sleep infinity
4026532218 mnt         2 27865 root    /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
4026532219 uts         2 27865 root    /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
4026532220 pid         2 27865 root    /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
4026532221 mnt         1 27977 _apt    sleep 365d
4026532222 uts         1 27977 _apt    sleep 365d
4026532223 pid         1 27977 _apt    sleep 365d
```

And the `httpbin` and `sidecar` containers seems to share the 'ipc' and 'net' namespaces:

```bash
# app container
$ sudo ls -l /proc/27865/ns
lrwxrwxrwx 1 root root 0 Oct 28 07:56 ipc -> 'ipc:[4026532159]'
lrwxrwxrwx 1 root root 0 Oct 28 07:56 mnt -> 'mnt:[4026532218]'
lrwxrwxrwx 1 root root 0 Oct 28 07:56 net -> 'net:[4026532162]'
lrwxrwxrwx 1 root root 0 Oct 28 07:56 pid -> 'pid:[4026532220]'
lrwxrwxrwx 1 root root 0 Oct 28 07:56 uts -> 'uts:[4026532219]'

# sidecar container
$ sudo ls -l /proc/27977/ns
lrwxrwxrwx 1 _apt systemd-journal 0 Oct 28 07:56 ipc -> 'ipc:[4026532159]'
lrwxrwxrwx 1 _apt systemd-journal 0 Oct 28 07:56 mnt -> 'mnt:[4026532221]'
lrwxrwxrwx 1 _apt systemd-journal 0 Oct 28 07:56 net -> 'net:[4026532162]'
lrwxrwxrwx 1 _apt systemd-journal 0 Oct 28 07:56 pid -> 'pid:[4026532223]'
lrwxrwxrwx 1 _apt systemd-journal 0 Oct 28 07:56 uts -> 'uts:[4026532222]'
```

### podman Pods

There is a Docker alternative OICD compatible Container engine called podman that can be managed pods of containers like Kubernetes.

First I start a pod wit some containers to test:

```bash
$ sudo podman pod create --name my_pod

$ sudo podman pod list

POD ID        NAME    STATUS   CREATED        INFRA ID      # OF CONTAINERS
0e0862e977e1  my_pod     Created  9 seconds ago  19e248401b83  1

$ sudo podman ps -a --pod

CONTAINER ID  IMAGE                 COMMAND  CREATED         STATUS   PORTS   NAMES               POD ID        PODNAME
19e248401b83  k8s.gcr.io/pause:3.5           13 seconds ago  Created          0e0862e977e1-infra  0e0862e977e1  my_pod

$ sudo podman run -dt --pod my_pod docker.io/curlimages/curl sleep 365d

$ sudo podman run -dt --pod my_pod docker.io/kennethreitz/httpbin

$ sudo podman ps -a --pod

CONTAINER ID  IMAGE                                  COMMAND               CREATED             STATUS                 PORTS   NAMES               POD ID        PODNAME
b4f923a0af26  k8s.gcr.io/pause:3.5                                         About a minute ago  Up About a minute ago          b49582203b1a-infra  b49582203b1a  my_pod
51f3c3ea6959  docker.io/curlimages/curl:latest       sleep 365d            About a minute ago  Up About a minute ago          gracious_hellman    b49582203b1a  my_pod
2a5b513a3089  docker.io/kennethreitz/httpbin:latest  gunicorn -b 0.0.0...  3 seconds ago       Up 3 seconds ago               laughing_villani    b49582203b1a  my_pod
```

Ten we can list the used namespaces:

```bash
$ sudo lsns
4026534263 net         4 25012 root             /pause
4026534335 mnt         1 25012 root             /pause
4026534336 uts         4 25012 root             /pause
4026534337 ipc         4 25012 root             /pause
4026534338 pid         1 25012 root             /pause
4026534340 mnt         1 25023 systemd-network  sleep 365d
4026534341 pid         1 25023 systemd-network  sleep 365d
4026534342 cgroup      1 25023 systemd-network  sleep 365d
4026534344 mnt         2 30514 root             /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
4026534345 pid         2 30514 root             /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
4026534346 cgroup      2 30514 root             /usr/bin/python3 /usr/local/bin/gunicorn -b 0.0.0.0:80 httpbin:app -k gevent
```

The containers shar the 'ipc', 'net', 'time', 'user' and 'uts' namespaces.

```bash
# httpbin container
sudo ls -l /proc/30514/ns
lrwxrwxrwx 1 root root 0 ápr   28 18:54 cgroup -> 'cgroup:[4026534346]'
lrwxrwxrwx 1 root root 0 ápr   28 18:54 ipc -> 'ipc:[4026534337]'
lrwxrwxrwx 1 root root 0 ápr   28 18:54 mnt -> 'mnt:[4026534344]'
lrwxrwxrwx 1 root root 0 ápr   28 18:54 net -> 'net:[4026534263]'
lrwxrwxrwx 1 root root 0 ápr   28 18:54 pid -> 'pid:[4026534345]'
lrwxrwxrwx 1 root root 0 ápr   28 18:54 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 ápr   28 18:54 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 ápr   28 18:54 uts -> 'uts:[4026534336]'


# curl container
sudo ls -l /proc/25023/ns
lrwxrwxrwx 1 systemd-network systemd-journal 0 ápr   28 18:54 cgroup -> 'cgroup:[4026534342]'
lrwxrwxrwx 1 systemd-network systemd-journal 0 ápr   28 18:54 ipc -> 'ipc:[4026534337]'
lrwxrwxrwx 1 systemd-network systemd-journal 0 ápr   28 18:54 mnt -> 'mnt:[4026534340]'
lrwxrwxrwx 1 systemd-network systemd-journal 0 ápr   28 18:54 net -> 'net:[4026534263]'
lrwxrwxrwx 1 systemd-network systemd-journal 0 ápr   28 18:54 pid -> 'pid:[4026534341]'
lrwxrwxrwx 1 systemd-network systemd-journal 0 ápr   28 18:54 time -> 'time:[4026531834]'
lrwxrwxrwx 1 systemd-network systemd-journal 0 ápr   28 18:54 user -> 'user:[4026531837]'
lrwxrwxrwx 1 systemd-network systemd-journal 0 ápr   28 18:54 uts -> 'uts:[4026534336]'
```



