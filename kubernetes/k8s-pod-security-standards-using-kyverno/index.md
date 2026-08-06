---
title: Pod Security Standards using Kyverno
url: https://devopstales.github.io/kubernetes/k8s-pod-security-standards-using-kyverno/
date: 2022-08-10
keywords: Kubernetes Secuity, Pod Security Policy, Pod Security, Pod Security Admission, Kyverno, mutating webhook, Admission Controller, seccomp
---


In this post I will show you how you can use Kyverno instal of Pod Security Admission.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### Why you need to use Kyverno instead of the Pod Security Admission

Probably your first question was why I want tho change the new built in solution to a third-party solution. There is pros and cons for bot solution but the changing of the Pod Security Policy to Pod Security Admission showed that the built in solution is not carved in stone. Previously I tested he new Pod Security Admission an I find the fallowing problems with it:

* **The tree predefined Pod Security Standards is not enough**, but there is no option to create your own.
* **No enforcement of Pod controllers**: This one is fairly major. For a profile in enforce mode, it will only block Pods emitted from a controller but NOT the actual controller itself. This results in, for example, ability to create a Deployment yet will silently block all the Pods it spawns.
* **Messages are not configurable**: Whatever message it generates in any of the modes is not configurable at all. Probably not a big deal.
* **Seeing audits is painful**: Being able to see audits involves digging into the API audit log. Setting up that log is a multi-step process, complex, not enabled by default, is disruptive if done retroactively, requires privileged access to the control plane, and the log cannot be viewed from inside the cluster.
* **Exemptions are very limited**: This is one of the biggest ones. Exemptions are limited to usernames, runtimeClasses, and Namespaces. Common use cases for exemption like for Pod or container name simply aren't there let alone more complex ones like ClusterRole. And in order to get even that you still have to configure the AdmissionConfiguration file, so see the above bullet for the difficulties that imposes.
* **Can't use in a pipeline**: PSA is engrained into the Kubernetes control plane, which means to test against it you have to actually submit something to the control plane. There's no standalone utility to know if, in advance, a given resource will work or not.

I know Pod Security Admission is a new thing. Just like with anything else, it takes time to mature a technology, but in production environment you cannot wait. So that is the reason I decided to use Kyverno instead of the Pod Security Admission.

### Perform node shell attack

I will use the node shell attack on a cluster where the Pod Security Admission is enabled for all namespaces.

First I need to install the kubernetes plugin:

```bash
curl -LO https://github.com/kvaps/kubectl-node-shell/raw/master/kubectl-node_shell
chmod +x ./kubectl-node_shell
sudo mv ./kubectl-node_shell /usr/local/bin/kubectl-node_shell
```

Basically, the node shell attack allows an attacker to get a shell as root on a node of the cluster by starting a privileged pod with access to host namespaces (`hostPID`, `hostIPC` and `hostNetwork`).

Some system component need this permissions to work so I will test in the `kube-system` namespace.

```bash
$ kubectl node-shell kind-control-plane -n kube-system

spawning "nsenter-wwsbcz" on "kind-control-plane"
If you don't see a command prompt, try pressing enter.
root@kind-control-plane:/# ls -la
total 60
drwxr-xr-x   1 root root 4096 Feb 24 16:31 .
drwxr-xr-x   1 root root 4096 Feb 24 16:31 ..
-rwxr-xr-x   1 root root    0 Feb 24 16:31 .dockerenv
lrwxrwxrwx   1 root root    7 Nov  2 20:43 bin -> usr/bin
drwxr-xr-x   2 root root 4096 Oct 11 08:39 boot
drwxr-xr-x  17 root root 4440 Feb 24 16:31 dev
drwxr-xr-x   1 root root 4096 Feb 24 16:31 etc
drwxr-xr-x   2 root root 4096 Oct 11 08:39 home
drwxr-xr-x   1 root root 4096 Feb 24 16:31 kind
lrwxrwxrwx   1 root root    7 Nov  2 20:43 lib -> usr/lib
lrwxrwxrwx   1 root root    9 Nov  2 20:43 lib32 -> usr/lib32
lrwxrwxrwx   1 root root    9 Nov  2 20:43 lib64 -> usr/lib64
lrwxrwxrwx   1 root root   10 Nov  2 20:43 libx32 -> usr/libx32
drwxr-xr-x   2 root root 4096 Nov  2 20:43 media
drwxr-xr-x   2 root root 4096 Nov  2 20:43 mnt
drwxr-xr-x   1 root root 4096 Jan 26 08:06 opt
dr-xr-xr-x 516 root root    0 Feb 24 16:31 proc
drwx------   1 root root 4096 Feb 24 17:07 root
drwxr-xr-x  11 root root  240 Feb 24 16:32 run
lrwxrwxrwx   1 root root    8 Nov  2 20:43 sbin -> usr/sbin
drwxr-xr-x   2 root root 4096 Nov  2 20:43 srv
dr-xr-xr-x  13 root root    0 Feb 24 16:31 sys
drwxrwxrwt   2 root root   40 Feb 24 17:30 tmp
drwxr-xr-x   1 root root 4096 Nov  2 20:43 usr
drwxr-xr-x  11 root root 4096 Feb 24 16:31 var
```

This is the problem wit Pod Security Admission it not flexible. Now lets try it with Kyverno.


### Deploy Kyverno

First deploy Kyverno without minimal configuration:

```bash
helm upgrade --install --wait --timeout 15m --atomic \
  --namespace kyverno --create-namespace \
  --repo https://kyverno.github.io/kyverno kyverno-policies \
  kyverno-policies --values - <<EOF
podSecurityStandard: restricted
validationFailureAction: enforce
EOF
```

The default kyverno policies for Pod Security Standard has the same problem. It ignore all request targeting the `kube-system`, `kube-public`, `kube-node-lease` and `kyverno` namespaces. So running the attack in `kube-system` namespace will succeed.

```bash
$ kubectl node-shell kind-control-plane -n kube-system

spawning "nsenter-wwsbcz" on "kind-control-plane"
If you don't see a command prompt, try pressing enter.
root@kind-control-plane:/# ls -la
total 60
drwxr-xr-x   1 root root 4096 Feb 24 16:31 .
drwxr-xr-x   1 root root 4096 Feb 24 16:31 ..
-rwxr-xr-x   1 root root    0 Feb 24 16:31 .dockerenv
lrwxrwxrwx   1 root root    7 Nov  2 20:43 bin -> usr/bin
drwxr-xr-x   2 root root 4096 Oct 11 08:39 boot
drwxr-xr-x  17 root root 4440 Feb 24 16:31 dev
drwxr-xr-x   1 root root 4096 Feb 24 16:31 etc
drwxr-xr-x   2 root root 4096 Oct 11 08:39 home
drwxr-xr-x   1 root root 4096 Feb 24 16:31 kind
lrwxrwxrwx   1 root root    7 Nov  2 20:43 lib -> usr/lib
lrwxrwxrwx   1 root root    9 Nov  2 20:43 lib32 -> usr/lib32
lrwxrwxrwx   1 root root    9 Nov  2 20:43 lib64 -> usr/lib64
lrwxrwxrwx   1 root root   10 Nov  2 20:43 libx32 -> usr/libx32
drwxr-xr-x   2 root root 4096 Nov  2 20:43 media
drwxr-xr-x   2 root root 4096 Nov  2 20:43 mnt
drwxr-xr-x   1 root root 4096 Jan 26 08:06 opt
dr-xr-xr-x 516 root root    0 Feb 24 16:31 proc
drwx------   1 root root 4096 Feb 24 17:07 root
drwxr-xr-x  11 root root  240 Feb 24 16:32 run
lrwxrwxrwx   1 root root    8 Nov  2 20:43 sbin -> usr/sbin
drwxr-xr-x   2 root root 4096 Nov  2 20:43 srv
dr-xr-xr-x  13 root root    0 Feb 24 16:31 sys
drwxrwxrwt   2 root root   40 Feb 24 17:30 tmp
drwxr-xr-x   1 root root 4096 Nov  2 20:43 usr
drwxr-xr-x  11 root root 4096 Feb 24 16:31 var
```

### With Kyverno I have the capability to create a custom solution.

We can add an exclude statement in our policies to allow requests coming from a user that belongs to the `system:nodes` group.

```bash
helm upgrade --install --wait --timeout 15m --atomic \
  --namespace kyverno --create-namespace \
  --repo https://kyverno.github.io/kyverno kyverno-policies \
  kyverno-policies --values - <<EOF
podSecurityStandard: restricted
validationFailureAction: enforce
background: false
policyExclude:
  disallow-capabilities:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  disallow-capabilities-strict:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
      - kind: Group
        name: system:serviceaccounts:kyverno
  disallow-host-namespaces:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  disallow-host-path:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  disallow-host-ports:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  disallow-host-process:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  disallow-privilege-escalation:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  disallow-privileged-containers:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  disallow-proc-mount:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  disallow-selinux:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  require-run-as-non-root-user:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  require-run-as-nonroot:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  restrict-apparmor-profiles:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  restrict-seccomp:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  restrict-seccomp-strict:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  restrict-sysctls:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
  restrict-volume-types:
    any:
    - subjects:
      - kind: Group
        name: system:nodes
      - kind: Group
        name: system:serviceaccounts:kube-system
EOF
```

### Attempt to run a node shell attack again

```bash
$ kubectl node-shell kind-control-plane -n kube-system

spawning "nsenter-dz6d2e" on "kind-control-plane"
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:resource Pod/kube-system/nsenter-dz6d2e was blocked due to the following policiesdisallow-capabilities-strict:
  require-drop-all: 'validation failure: Containers must drop `ALL` capabilities.'
disallow-host-namespaces:
  host-namespaces: 'validation error: Sharing the host namespaces is disallowed. The
    fields spec.hostNetwork, spec.hostIPC, and spec.hostPID must be unset or set to
    `false`. Rule host-namespaces failed at path /spec/hostNetwork/'
disallow-privilege-escalation:
  privilege-escalation: 'validation error: Privilege escalation is disallowed. The
    fields spec.containers[*].securityContext.allowPrivilegeEscalation, spec.initContainers[*].securityContext.allowPrivilegeEscalation,
    and spec.ephemeralContainers[*].securityContext.allowPrivilegeEscalation must
    be set to `false`. Rule privilege-escalation failed at path /spec/containers/0/securityContext/allowPrivilegeEscalation/'
disallow-privileged-containers:
  privileged-containers: 'validation error: Privileged mode is disallowed. The fields
    spec.containers[*].securityContext.privileged and spec.initContainers[*].securityContext.privileged
    must be unset or set to `false`. Rule privileged-containers failed at path /spec/containers/0/securityContext/privileged/'
require-run-as-nonroot:
  run-as-non-root: 'validation error: Running as root is not allowed. Either the field
    spec.securityContext.runAsNonRoot must be set to `true`, or the fields spec.containers[*].securityContext.runAsNonRoot,
    spec.initContainers[*].securityContext.runAsNonRoot, and spec.ephemeralContainers[*].securityContext.runAsNonRoot
    must be set to `true`. Rule run-as-non-root[0] failed at path /spec/securityContext/runAsNonRoot/.
    Rule run-as-non-root[1] failed at path /spec/containers/0/securityContext/runAsNonRoot/.'
restrict-seccomp-strict:
  check-seccomp-strict: 'validation error: Use of custom Seccomp profiles is disallowed.
    The fields spec.securityContext.seccompProfile.type, spec.containers[*].securityContext.seccompProfile.type,
    spec.initContainers[*].securityContext.seccompProfile.type, and spec.ephemeralContainers[*].securityContext.seccompProfile.type
    must be set to `RuntimeDefault` or `Localhost`. Rule check-seccomp-strict[0] failed
    at path /spec/securityContext/seccompProfile/. Rule check-seccomp-strict[1] failed
    at path /spec/containers/0/securityContext/seccompProfile/.'
```

