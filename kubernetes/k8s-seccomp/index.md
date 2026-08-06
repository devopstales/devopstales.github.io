---
title: Hardening Kubernetes with seccomp
url: https://devopstales.github.io/kubernetes/k8s-seccomp/
date: 2021-09-03
keywords: kubernetes deployment, seccomp, docker, containerd, CRI-O, Kubernetes, BPF
---


In this post I will attempt to demystify the relationship of `seccomp` and Kubernetes This first part will look at containers and pods.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

With Kubernetes version v1.22 there is a new alpha feature that provides a way to use the `RuntimeDefault` as the defaut seccomp profile insted of `Unconfined`. By default, when Kubernetes makes a new container it creates with `Unconfined` seccomp profile. This means that seccomp filtering is disabled.

### Wthat is seccomp profile?
[Seccomp (Secure Computing)](https://en.wikipedia.org/wiki/Seccomp) is a feature in the Linux kernel. It allow to create profiles to filter [system calls](https://en.wikipedia.org/wiki/System_call). Usage of seccomp profiles on containers reduces the chance that a Linux kernel vulnerability will be exploited. All container runtimes ship with a default seccomp profile. The problem come when we using Kubernetes, beasuse Kubernetes use `Unconfined` as default and disables seccomp filtering.

For example Docker's default seccomp profile disables approximately 44 system calls of the 300+ currently availble.

### Test Seccomp profile.
For the test I will use `amicontained` to inspection tool. First test in a simple docker.

```bash
docker run --rm -it r.j3ss.co/amicontained bash
Container Runtime: docker
Has Namespaces:
	pid: true
	user: false
AppArmor Profile: docker-default (enforce)
Capabilities:
	BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: filtering
Blocked Syscalls (60):
	SYSLOG SETPGID SETSID USELIB USTAT SYSFS VHANGUP PIVOT_ROOT _SYSCTL ACCT SETTIMEOFDAY MOUNT UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL IOPERM CREATE_MODULE INIT_MODULE DELETE_MODULE GET_KERNEL_SYMS QUERY_MODULE QUOTACTL NFSSERVCTL GETPMSG PUTPMSG AFS_SYSCALL TUXCALL SECURITY LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MEMPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY KEYCTL MIGRATE_PAGES UNSHARE MOVE_PAGES PERF_EVENT_OPEN FANOTIFY_INIT NAME_TO_HANDLE_AT OPEN_BY_HANDLE_AT SETNS PROCESS_VM_READV PROCESS_VM_WRITEV KCMP FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD PKEY_MPROTECT PKEY_ALLOC PKEY_FREE
Looking for Docker.sock
```

As you can see with the default Docker secom profile 60 Syscalls are being blocked. Now test wit default Kubernetes config on docker.

```bash
kubectl run -it bash --image=r.j3ss.co/amicontained --restart=Never bash
Container Runtime: docker
Has Namespaces:
 pid: true
 user: false
AppArmor Profile: unconfined
Capabilities:
 BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: disabled
Blocked Syscalls (21):
 MSGRCV SYSLOG SETSID VHANGUP PIVOT_ROOT ACCT SETTIMEOFDAY SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME INIT_MODULE DELETE_MODULE LOOKUP_DCOOKIE KEXEC_LOAD FANOTIFY_INIT OPEN_BY_HANDLE_AT FINIT_MODULE KEXEC_FILE_LOAD BPF
Looking for Docker.sock
```

In the output above you can see that seccomp is disabled and that 21 syscalls are being blocked. Now test wit default Kubernetes config on rke2 (containerd).



```bash
kubectl run -it bash --image=r.j3ss.co/amicontained --restart=Never bash
Container Runtime: kube
Has Namespaces:
	pid: true
	user: false
AppArmor Profile: system_u:system_r:container_t:s0:c575,c847
Capabilities:
	BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: disabled
Blocked Syscalls (22):
	SYSLOG SETPGID SETSID VHANGUP PIVOT_ROOT ACCT SETTIMEOFDAY UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME INIT_MODULE DELETE_MODULE LOOKUP_DCOOKIE KEXEC_LOAD FANOTIFY_INIT OPEN_BY_HANDLE_AT FINIT_MODULE KEXEC_FILE_LOAD BPF
Looking for Docker.sock
pod default/bash terminated (Error)
```

The `containerd` is similar then docker so lets test with `CRI-O`.

```bash
kubectl run -it bash --image=bash --restart=Never bash
# apk add curl
# curl -LO k8s.work/amicontained
# chmod +x amicontained
# ./amicontained
Container Runtime: kube
Has Namespaces:
	pid: true
	user: false
AppArmor Profile: system_u:system_r:spc_t:s0
Capabilities:
	BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service
Seccomp: disabled
Blocked Syscalls (22):
	MSGRCV SYSLOG SETSID VHANGUP PIVOT_ROOT ACCT SETTIMEOFDAY UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME INIT_MODULE DELETE_MODULE LOOKUP_DCOOKIE KEXEC_LOAD FANOTIFY_INIT OPEN_BY_HANDLE_AT FINIT_MODULE KEXEC_FILE_LOAD BPF
Looking for Docker.sock
```

### Enable RuntimeDefault seccomp profile

Enable in local kubelet config:

```bash
nano /var/lib/kubelet/config.yaml
...
--feature-gates="...,SeccompDefault=true"
--seccomp-default RuntimeDefault

systemctl restart kubelet
```

Enable in running kubelet config:

```bash
kubectl edit cm kubelet-config-1.22 -n kub-system
...
- --feature-gates="...,SeccompDefault=true"
- --seccomp-default RuntimeDefault
```

Then test:

```bash
kubectl run -it bash --image=r.j3ss.co/amicontained --restart=Never bash
Container Runtime: docker
Has Namespaces:
 pid: true
 user: false
AppArmor Profile: unconfined
Capabilities:
 BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: filtering
Blocked Syscalls (61):
 MSGRCV PTRACE SYSLOG SETSID USELIB USTAT SYSFS VHANGUP PIVOT_ROOT _SYSCTL ACCT SETTIMEOFDAY MOUNT UMOUNT2 SWAPON SWAPOFF REBOOT SETHOSTNAME SETDOMAINNAME IOPL IOPERM CREATE_MODULE INIT_MODULE DELETE_MODULE GET_KERNEL_SYMS QUERY_MODULE QUOTACTL NFSSERVCTL GETPMSG PUTPMSG AFS_SYSCALL TUXCALL SECURITY LOOKUP_DCOOKIE CLOCK_SETTIME VSERVER MBIND SET_MEMPOLICY GET_MEMPOLICY KEXEC_LOAD ADD_KEY REQUEST_KEY KEYCTL MIGRATE_PAGES UNSHARE MOVE_PAGES PERF_EVENT_OPEN FANOTIFY_INIT NAME_TO_HANDLE_AT OPEN_BY_HANDLE_AT SETNS PROCESS_VM_READV PROCESS_VM_WRITEV KCMP FINIT_MODULE KEXEC_FILE_LOAD BPF USERFAULTFD PKEY_MPROTECT PKEY_ALLOC PKEY_FREE
Looking for Docker.sock
```

### Customizing a Profile

One way to write `seccomp` filter is to use Berkeley packet filter (BPF) language. Using this language isn't really simple or convenient. We can write JSON that is compiled into profile by `libseccomp`. 

If you were to create a profile to allow a container to execute a ping against a website, you can use `strace` command to find the syscalls it makes:

```bash
strace -fqc ping -c 20 www.google.com% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 29.55    0.000078           4        20        11 openat
 14.02    0.000037           9         4         4 socket
 11.74    0.000031           3        12           mprotect
  6.06    0.000016           2         7           read
  5.68    0.000015           1        17           mmap
  5.68    0.000015           3         5           capget
  4.92    0.000013          13         1           munmap
  3.79    0.000010           1         9           fstat
  3.41    0.000009           9         1           write
  3.41    0.000009           1         9           close
  2.65    0.000007           2         3           brk
  2.65    0.000007           4         2           prctl
  2.27    0.000006           3         2           getuid
  1.52    0.000004           4         1           setuid
  1.52    0.000004           4         1           capset
  1.14    0.000003           3         1           geteuid
  0.00    0.000000           0         9         9 access
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         3           fcntl
  0.00    0.000000           0         1           arch_prctl
------ ----------- ----------- --------- --------- ----------------
100.00    0.000264                   109        24 total
```

A nothe solution is a tool called [zaz](https://github.com/pjbgf/zaz) created by [Paulo Gomes](https://github.com/pjbgf) That generate a seccomp prifile for you with the minimum system calls:

```bash
zaz seccomp docker alpine "ping -c5 8.8.8.8"
```

A basic seccomp has three key elements: the `defaultAction`, the `architectures` (or `archMap`) and the `syscalls`: 

```json
mkdir /var/lib/kubelet/seccomp
nano /var/lib/kubelet/seccomp/sample.json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "arch_prctl",
                "sched_yield",
                "futex",
                "write",
                "mmap",
                "exit_group",
                "madvise",
                "rt_sigprocmask",
                "getpid",
                "gettid",
                "tgkill",
                "rt_sigaction",
                "read",
                "getpgrp"
            ],
            "action": "SCMP_ACT_ALLOW",
   "args": [],
   "comment": "",
   "includes": {},
   "excludes": {}
        }
    ]
}
```

The `defaultAction` is `SCMP_ACT_ERRNO` which will block the execution of any system call. The we list the `syscalls` what we want to whitelist. 

### The different types of actions

Below is a list of all the different types of actions and what they do:

```bash
SCMP_ACT_KILL_THREAD (or SCMP_ACT_KILL)
Does not execute the syscall and terminate the thread that attempted making the call. Note that depending on the application being enforced (i.e. multi-threading) and its error handling, syscalls blocked using this action may do so silently which may result in side effects on the overall application.

SCMP_ACT_TRAP
Does not execute the syscall. The kernel will send a thread-directed SIGSYS signal to the thread that attempted making the call.

SCMP_ACT_ERRNO
Does not execute the syscall, returns error instead. Note that depending on the error handling of the application being enforced, syscalls blocked using this action may do so silently which may result in side effects on the overall application.

SCMP_ACT_TRACE
The decision on whether or not to execute the syscall will come from a tracer. If no tracer is present behaves like SECCOMP_RET_ERRNO.
This can be used to automate profile generation and also can be used to change the syscall being made. Not recommended when trying to enforce seccomp to line of business applications.

SCMP_ACT_ALLOW
Executes the syscall.

SCMP_ACT_LOG (since Linux 4.14)
Executes the syscall. Useful for running seccomp in "complain-mode", logging the syscalls that are mapped (or catch-all) and not blocking their execution. It can be used together with other action types to provide an allow and deny list approach.

SCMP_ACT_KILL_PROCESS (since Linux 4.14)
Does not execute the syscall and terminates the entire process with a core dump. Very useful when automating the profile generation.
```

### Configure on a pod

With Kubernetes version v1.19 Seccomp Profile for a Container is GA. The sintax looks like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: some-pod
  labels:
    app: some-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: sample.json
  containers:
    ...
```

Valid options for `type` include `RuntimeDefault`, `Unconfined`, and `Localhost`. Here is an example that sets the Seccomp profile to the node's container runtime default profile:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: some-pod
  labels:
    app: some-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    ...
```

This Configuration is the same then setting `SeccompDefault=true` in kubelet config.

### Add audit profile

Since linux kernel 4.14 it is now possible to define parts of your profile to run in audit mode, logging into syslog all the system calls you want without blocking them. To do that you can use the action SCMT_ACT_LOG:

```json
nano /var/lib/kubelet/seccomp/audit.json
{
    "defaultAction": "SCMP_ACT_LOG",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "arch_prctl",
                "sched_yield",
                "futex",
                "write",
                "mmap",
                "exit_group",
                "madvise",
                "rt_sigprocmask",
                "getpid",
                "gettid",
                "tgkill",
                "rt_sigaction",
                "read",
                "getpgrp"
            ],
            "action": "SCMP_ACT_ALLOW"
        },
        {
            "names": [
                "add_key",
                "keyctl",
                "ptrace"
            ],
            "action": "SCMP_ACT_ERRNO"
        }
    ]
}
```

```bash
~ $ tail /var/log/syslog

Nov 25 19:38:18 kernel: [461698.749294] audit: ... syscall=21 compat=0 ip=0x7ff8f8412d5b code=0x7ffc0000    # access
Nov 25 19:38:18 kernel: [461698.749306] audit: ... syscall=257 compat=0 ip=0x7ff8f8412ec8 code=0x7ffc0000   # openat
Nov 25 19:38:18 kernel: [461698.749315] audit: ... syscall=5 compat=0 ip=0x7ff8f8412c99 code=0x7ffc0000     # fstat
Nov 25 19:38:18 kernel: [461698.749317] audit: ... syscall=9 compat=0 ip=0x7ff8f84130e6 code=0x7ffc0000     # mmap
Nov 25 19:38:18 kernel: [461698.749323] audit: ... syscall=3 compat=0 ip=0x7ff8f8412d8b code=0x7ffc0000     # close
```

### Set capabilities for a Container

With version 1.22 you should be able to change the sysctl in the security context of your pod manifests, allowing containers that are running as unprivileged users to bind low ports.

```yaml
securityContext:
     sysctls:
     - name: net.ipv4.ip_unprivileged_port_start
          value: "1"
```

### Final Words

Whatever you define in your seccomp profile, the kernel will enforce it. Even if that is not what you want. For example, if you block access to calls such as exit or exit_group your container may not be able to exit and it could trap the container in an exit loop indefinitely. Leading to high CPU usage of your cluster.

