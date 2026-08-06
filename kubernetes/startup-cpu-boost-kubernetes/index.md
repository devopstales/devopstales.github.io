---
title: Accelerating Application Startup in Kubernetes with In-Place Pod Resize
url: https://devopstales.github.io/kubernetes/startup-cpu-boost-kubernetes/
date: 2026-03-30
keywords: Kubernetes startup optimization, In-Place Pod Resize, Startup CPU Boost, Java performance Kubernetes, pod resource scaling, vertical scaling
---


Learn how to accelerate Java and other resource-heavy application startups in Kubernetes using In-Place Pod Resize and the Kube Startup CPU Boost controller—without permanent over-provisioning.

<!--more-->

## The Problem: CPU-Intensive Application Startup

If you've ever deployed Java applications (or other resource-heavy workloads) to Kubernetes, you know the pain: your application needs significant CPU during startup for class loading, JIT compilation, and initialization, but runs comfortably with modest resources once ready.

The traditional approaches all have drawbacks:

- **Over-provisioning**: Set high CPU requests/limits permanently, wasting resources and money
- **Under-provisioning**: Keep resources low, but accept slow startup times and potential timeouts
- **No limits**: Remove CPU limits entirely, risking noisy neighbor problems

There's now a better solution: **temporary CPU boost during startup** with automatic reversion to normal resources once the application is ready.

## Enter In-Place Pod Resize

Kubernetes 1.35 graduated **In-Place Pod Resize** to GA, allowing you to modify CPU and memory requests/limits without restarting pods. This feature, combined with the **Kube Startup CPU Boost** controller, enables a powerful pattern: automatically grant extra CPU during startup, then revert to baseline once the application signals readiness.

### How It Works

1. Pod starts with baseline resources (e.g., 200m CPU)
2. Startup CPU Boost controller detects the new pod
3. Controller temporarily increases CPU (e.g., +50% or fixed amount)
4. Application starts faster with additional CPU headroom
5. Once readiness probe succeeds, controller reverts to original resources
6. Pod continues running with normal resource allocation

No pod restart. No manual intervention. No permanent over-provisioning.

## Prerequisites

### Enable the Feature Gate

In-Place Pod Resize must be enabled in your cluster. For development clusters like Minikube:

```bash
minikube start --feature-gates=InPlacePodVerticalScaling=true
```

For production clusters, ensure the `InPlacePodVerticalScaling` feature gate is enabled in your kubelet configuration.

### Install the Controller

Deploy the Kube Startup CPU Boost controller via Helm:

```bash
helm repo add kube-startup-cpu-boost https://google.github.io/kube-startup-cpu-boost
helm repo update
helm install kube-startup-cpu-boost kube-startup-cpu-boost/kube-startup-cpu-boost \
  --namespace kube-startup-cpu-boost-system --create-namespace
```

The controller runs in the `kube-startup-cpu-boost-system` namespace and watches for `StartupCPUBoost` resources.

## Configuration

### Step 1: Define the StartupCPUBoost Resource

Create a `StartupCPUBoost` custom resource that targets your workload:

```yaml
apiVersion: autoscaling.x-k8s.io/v1alpha1
kind: StartupCPUBoost
metadata:
  name: java-app-boost
  namespace: production
spec:
  selector:
    matchExpressions:
    - key: app.kubernetes.io/name
      operator: In
      values: ["my-java-app"]
  resourcePolicy:
    containerPolicies:
    - containerName: my-java-app
      percentageIncrease:
        value: 50
  durationPolicy:
    podCondition:
      type: Ready
      status: "True"
```

This configuration:
- Targets pods with label `app.kubernetes.io/name: my-java-app`
- Increases CPU by 50% during startup
- Maintains the boost until the pod's `Ready` condition is `True`

### Alternative: Fixed Resource Allocation

Instead of a percentage increase, you can specify exact resources:

```yaml
spec:
  resourcePolicy:
    containerPolicies:
    - containerName: my-java-app
      fixedResources:
        requests: "500m"
        limits: "2"
```

### Step 2: Configure Your Deployment

Your deployment must specify the `resizePolicy` to indicate that CPU changes don't require a restart:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-java-app
  namespace: production
  labels:
    app.kubernetes.io/name: my-java-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: my-java-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-java-app
    spec:
      containers:
      - name: my-java-app
        image: myregistry/my-java-app:latest
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 1Gi
        resizePolicy:
        - resourceName: "cpu"
          restartPolicy: "NotRequired"
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

**Critical**: The `resizePolicy` with `restartPolicy: "NotRequired"` is what enables in-place resource changes. Without it, the pod would need to restart when resources are modified.

## How Resources Change During the Lifecycle

### During Startup Boost

With a 50% percentage increase policy:

| Resource | Baseline | During Boost |
|----------|----------|--------------|
| CPU Request | 200m | 300m |
| CPU Limit | 500m | Removed* |

*By default, CPU limits are removed during boost to prevent throttling. To retain limits, set the environment variable `REMOVE_LIMITS=false` in the controller manager deployment.

### After Readiness

Once the readiness probe succeeds:

| Resource | Value |
|----------|-------|
| CPU Request | 200m (reverted) |
| CPU Limit | 500m (restored) |

## Monitoring the Boost

The controller integrates with Prometheus for monitoring. You can track:

- Current CPU requests vs. baseline
- Duration of boost periods
- Number of active boosts

Example Prometheus query to visualize CPU changes:

```promql
kube_pod_container_resource_requests{resource="cpu", container="my-java-app"}
```

You'll see the request value spike during pod creation, then return to baseline once the application becomes ready.

## Real-World Performance Impact

In testing with typical Spring Boot applications:

- **Startup time reduction**: 30-50% faster startup with 50% CPU boost
- **Resource efficiency**: No permanent over-provisioning
- **Reliability**: Fewer startup timeouts and failed health checks

The CPU usage pattern typically shows:
1. High CPU spike during class loading and initialization
2. Gradual decrease as application stabilizes
3. Steady-state operation at much lower CPU utilization

This pattern perfectly matches the startup boost approach: give extra resources when needed, remove them when not.

## Alternatives and Considerations

### Other Approaches

1. **No CPU Limits**: Simply remove limits entirely. This avoids throttling but doesn't help with scheduling priority and can cause noisy neighbor issues.

2. **Java-Specific Optimizations**:
   - **GraalVM Native Images**: Compile to native code, dramatically reducing startup time and CPU needs
   - **CRaC (Coordinated Restore at Checkpoint)**: Pre-warm applications and checkpoint state

3. **Policy-Based Mutation**: Tools like Kyverno can mutate resource requests, but require manual updates when the In-Place Pod Resize feature graduated to GA.

4. **Vertical Pod Autoscaler (VPA)**: VPA now supports in-place resize but doesn't yet have a dedicated startup boost policy (expected in future versions).

### When to Use Startup CPU Boost

**Good fit:**
- Java/Kotlin applications with long startup times
- Applications with distinct startup vs. steady-state resource needs
- Workloads where startup timeouts are a concern
- Teams wanting to optimize resource utilization

**Consider alternatives:**
- Applications with minimal startup overhead
- Workloads already using native compilation
- Clusters without In-Place Pod Resize enabled

## Conclusion

The combination of In-Place Pod Resize and Kube Startup CPU Boost solves a real operational problem: how to give applications the resources they need during startup without wasting capacity during steady-state operation.

The beauty of this approach is its elegance:
- **Declarative**: Define boost policies as Kubernetes resources
- **Automatic**: No manual intervention required
- **Temporary**: Resources automatically revert when ready
- **Efficient**: No permanent over-provisioning

As more clusters enable In-Place Pod Resize and tooling matures, this pattern will likely become the standard approach for resource-intensive application startups in Kubernetes.

## References

- [Kubernetes In-Place Pod Resize Documentation](https://kubernetes.io/docs/concepts/workloads/pods/in-place-pod-resize/)
- [Kube Startup CPU Boost GitHub](https://github.com/google/kube-startup-cpu-boost)
- [Kubernetes Feature Gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)

