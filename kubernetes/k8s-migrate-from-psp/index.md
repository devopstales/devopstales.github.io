---
title: Kubernetes: How to migrate Pod Security Policy to Pod Security Admission?
url: https://devopstales.github.io/kubernetes/k8s-migrate-from-psp/
date: 2022-08-24
keywords: Kubernetes Secuity, best Practices, Pod Security Policy, Pod Security, Pod Security Admission
---


With the release of Kubernetes v1.25, Pod Security admission has now entered to stable and PodSecurityPolicy is removed. In this article, I will show you how you can migrate to the new Pod Security Admission.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

### Requirements and limitations

* `PodSecurity` is available in k8s versions 1.23 and later.
* `PodSecurity` doesn't terminate Pods that are already running on your nodes, even if they violate the applied policy.
* `PodSecurity` doesn't mutate fields. If you use any mutating fields in your PodSecurityPolicy, modify your Pod spec to ensure that those fields are present when you deploy the workloads.

### Configure the PodSecurity admission controller in your cluster

In a nutshell `PodSecurity` enforces Pod Security Standards at the namespace level. So you need to chose one predefined policies fo every namespaces. The following policies are available:

* `Restricted`: Most restrictive policy. Complies with Pod hardening best practices.
* `Baseline`: Minimally restrictive policy that prevents known privilege escalations. Allows all default values for fields in Pod specifications.
* `Privileged`: Unrestricted policy that allows anything, including known privilege escalations. Apply this policy with caution.

### Eliminate mutating PodSecurityPolicies, if your cluster has any set up.

* Clone all mutating PSPs into a non-mutating version.
* Update all ClusterRoles authorizing use of those mutating PSPs to also authorize use of the non-mutating variant.
* Watch for Pods using the mutating PSPs and work with code owners to migrate to valid, non-mutating resources.
* Delete mutating PSPs.

You can start by eliminating the fields that are purely mutating, and don't have any bearing on the validating policy:

* `.spec.defaultAllowPrivilegeEscalation`
* `.spec.runtimeClass.defaultRuntimeClassName`
* `.metadata.annotations['seccomp.security.alpha.kubernetes.io/defaultProfileName']`
* `.metadata.annotations['apparmor.security.beta.kubernetes.io/defaultProfileName']`
* `.spec.defaultAddCapabilities` - Although technically a mutating & validating field, these should be merged into `.spec.allowedCapabilities` which performs the same validation without mutation.

There are several fields in PodSecurityPolicy that are not covered by the Pod Security Standards:

* `.spec.allowedHostPaths`
* `.spec.allowedFlexVolumes`
* `.spec.allowedCSIDrivers`
* `.spec.forbiddenSysctls`
* `.spec.runtimeClass`
* `.spec.requiredDropCapabilities` - Required to drop ALL for the Restricted profile.
* `.spec.seLinux` - (Only mutating with the MustRunAs rule) required to enforce the SELinux requirements of the Baseline & Restricted profiles.
* `.spec.runAsUser` - (Non-mutating with the RunAsAny rule) required to enforce RunAsNonRoot for the Restricted profile.
* `.spec.allowPrivilegeEscalation` - (Only mutating if set to false) required for the Restricted profile

Identify pods running under the original PSP. This can be done using the `kubernetes.io/psp` annotation. For example, using `kubectl`:

```bash
PSP_NAME="original" # Set the name of the PSP you're checking for
kubectl get pods --all-namespaces -o jsonpath="{range .items[?(@.metadata.annotations.kubernetes\.io\/psp=='$PSP_NAME')]}{.metadata.namespace} {.metadata.name}{'\n'}{end}"
```

Compare these running pods against the original pod spec to determine whether `PodSecurityPolicy` has modified the pod. For pods created by a workload resource you can compare the pod with the PodTemplate in the controller resource. 

Create the new `PodSecurityPolicies`. If any Roles or ClusterRoles are granting use on all PSPs this could cause the new PSPs to be used instead of their mutating counter-parts.

Update your authorization to grant access to the new PSPs. In RBAC this means updating any Roles or ClusterRoles that grant the use permision on the original PSP to also grant it to the updated PSP.

Verify: after some soak time, rerun the command from the begging to see if any pods are still using the original PSPs. Note that pods need to be recreated after the new policies have been rolled out before they can be fully verified.

Once you have verified that the original PSPs are no longer in use, you can delete them.

### Apply a Pod Security Standards in dry-run mode fo each namespace

Apply the Restricted policy in dry-run mode:

```bash
kubectl label --dry-run=server --overwrite ns $NAMESPACE \
    pod-security.kubernetes.io/enforce=restricted
```

If a Pod in the namespace violates the Restricted policy, the output is similar to the following:

```bash
Warning: existing pods in namespace "NAMESPACE" violate the new PodSecurity enforce level "restricted:latest"
namespace/NAMESPACE labeled
```

If the Restricted policy displays a warning, modify your Pods to fix the violation and try the command again. Alternatively, try the less restrictive Baseline policy in the following step.

```bash
kubectl label --dry-run=server --overwrite ns NAMESPACE \
    pod-security.kubernetes.io/enforce=baseline
```

> Caution: You can optionally use the Privileged policy, which has no restrictions. Before using the Privileged policy, ensure that you trust all workloads and users that have access to the namespace. The Privileged policy allows known privilege escalations, but may be required for certain privileged system workloads.

### Enforce the policy on a namespace

When you identify the policy that works for a namespace, apply the policy to the namespace in `enforce` mode:

```bash
kubectl label --overwrite ns $NAMESPACE \
    pod-security.kubernetes.io/enforce=restricted

```

### Review namespace creation processes

Updating the existing namespace is one thing, but you need to create the new namespaces wit  Pod Security Admission. You can use Kyverno to automaticle add the necessary label to the namespace at creation:

```bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-psa-labels
  annotations:
    policies.kyverno.io/title: Add PSA Labels
    policies.kyverno.io/category: Pod Security Admission
    policies.kyverno.io/severity: medium
    kyverno.io/kyverno-version: 1.7.1
    policies.kyverno.io/minversion: 1.6.0
    kyverno.io/kubernetes-version: "1.24"
    policies.kyverno.io/subject: Namespace
    policies.kyverno.io/description: >-
      Pod Security Admission (PSA) can be controlled via the assignment of labels
      at the Namespace level which define the Pod Security Standard (PSS) profile
      in use and the action to take. If not using a cluster-wide configuration
      via an AdmissionConfiguration file, Namespaces must be explicitly labeled.
      This policy assigns the labels `pod-security.kubernetes.io/enforce=baseline`
      and `pod-security.kubernetes.io/warn=restricted` to all new Namespaces if
      those labels are not included.
spec:
  rules:
  - name: add-baseline-enforce-restricted-warn
    match:
      any:
      - resources:
          kinds:
          - Namespace
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            +(pod-security.kubernetes.io/enforce): baseline
            +(pod-security.kubernetes.io/warn): restricted
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-privileged-profile
  annotations:
    policies.kyverno.io/title: Deny Privileged Profile
    policies.kyverno.io/category: Pod Security Admission
    policies.kyverno.io/severity: medium
    kyverno.io/kyverno-version: 1.7.1
    policies.kyverno.io/minversion: 1.6.0
    kyverno.io/kubernetes-version: "1.24"
    policies.kyverno.io/subject: Namespace
    policies.kyverno.io/description: >-
      When Pod Security Admission (PSA) is enforced at the cluster level
      via an AdmissionConfiguration file which defines a default level at
      baseline or restricted, setting of a label at the `privileged` profile
      will effectively cause unrestricted workloads in that Namespace, overriding
      the cluster default. This may effectively represent a circumvention attempt
      and should be closely controlled. This policy ensures that only those holding
      the cluster-admin ClusterRole may create Namespaces which assign the label
      `pod-security.kubernetes.io/enforce=privileged`.
spec:
  validationFailureAction: audit
  rules:
  - name: check-privileged
    match:
      any:
      - resources:
          kinds:
            - Namespace
          selector:
            matchLabels:
              pod-security.kubernetes.io/enforce: privileged
    exclude:
      any:
      - clusterRoles:
        - cluster-admin
    validate:
      message: Only cluster-admins may create Namespaces that allow setting the privileged level.
      deny: {}
```

[In a previous Post](/kubernetes/k8s-pod-security-standards-using-kyverno/) I showed you how you can use Kyverno instal of Pod Security Admission.

### Disable the PodSecurityPolicy feature on your cluster

On all master edit tha api-server config:

```bash
nano /etc/kubernetes/manifests/kube-apiserver.yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
    ...
#    - --enable-admission-plugins="NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,PodSecurityPolicy"
    - --enable-admission-plugins="NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook"
```

Restart the api-server pod.

