---
title: Kubernetes Multicluster with Cilium Cluster Mesh
url: https://devopstales.github.io/kubernetes/cluster-mesh-with-cilium/
date: 2023-07-15
keywords: Kubernetes, k8s, Load Balancer, kubernetes deployment, kubernetes ingress, Cilium, Cluster Mesh, Multi-Cluster, clustermesh
---


In this tutorial I will show you how to install Cilium on multiple Kubernetes clusters and connect those clusters with Cluster Mesh.

<!--more-->

### Bootstrap kind clusters

```yaml
nano kind-c1-config.yaml 
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
networking:
  disableDefaultCNI: true
  podSubnet: "10.0.0.0/16"
  serviceSubnet: "10.1.0.0/16"
```

```yaml
nano kind-c2-config.yaml 
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
networking:
  disableDefaultCNI: true
  podSubnet: "10.2.0.0/16"
  serviceSubnet: "10.3.0.0/16"
```

```bash
kind create cluster --name c1 --config kind-c1-config.yaml 
kind create cluster --name c2 --config kind-c2-config.yaml
```

### Install Cilium

```bash
helm repo add cilium https://helm.cilium.io/

kubectx kind-c1

helm install cilium cilium/cilium \
   --namespace kube-system \
   --set nodeinit.enabled=true \
   --set kubeProxyReplacement=partial \
   --set hostServices.enabled=false \
   --set externalIPs.enabled=true \
   --set nodePort.enabled=true \
   --set hostPort.enabled=true \
   --set cluster.name=c1 \
   --set cluster.id=1
```

```bash
cilium status

    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet              cilium             Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium             Running: 2
                       cilium-operator    Running: 2
Cluster Pods:          4/4 managed by Cilium
Helm chart version:    1.14.0
Image versions         cilium             quay.io/cilium/cilium:v1.14.0@sha256:5a94b561f4651fcfd85970a50bc78b201cfbd6e2ab1a03848eab25a82832653a: 2
                       cilium-operator    quay.io/cilium/operator-generic:v1.14.0@sha256:3014d4bcb8352f0ddef90fa3b5eb1bbf179b91024813a90a0066eb4517ba93c9: 2
```

```bash
kubectx kind-c2

kubectl --context=kind-c1 get secret -n kube-system cilium-ca -o yaml | \
  kubectl --context kind-c2 create -f -

helm install cilium cilium/cilium \
   --namespace kube-system \
   --set nodeinit.enabled=true \
   --set kubeProxyReplacement=partial \
   --set hostServices.enabled=false \
   --set externalIPs.enabled=true \
   --set nodePort.enabled=true \
   --set hostPort.enabled=true \
   --set cluster.name=c2 \
   --set cluster.id=2
```

```bash
cilium status

    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium             Desired: 2, Ready: 2/2, Available: 2/2
Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium             Running: 2
                       cilium-operator    Running: 2
Cluster Pods:          4/4 managed by Cilium
Helm chart version:    1.14.0
Image versions         cilium             quay.io/cilium/cilium:v1.14.0@sha256:5a94b561f4651fcfd85970a50bc78b201cfbd6e2ab1a03848eab25a82832653a: 2
                       cilium-operator    quay.io/cilium/operator-generic:v1.14.0@sha256:3014d4bcb8352f0ddef90fa3b5eb1bbf179b91024813a90a0066eb4517ba93c9: 2
```


### Enable Cilium Cluster Mesh

```bash
cilium clustermesh enable --context kind-c1 --service-type NodePort
cilium clustermesh enable --context kind-c2 --service-type NodePort
```

Connect the clusters:

```bash
cilium clustermesh connect --context kind-c1 --destination-context kind-c2

cilium clustermesh status --context kind-c1 --wait
⚠️  Service type NodePort detected! Service may fail when nodes are removed from the cluster!
✅ Service "clustermesh-apiserver" of type "NodePort" found
✅ Cluster access information is available:
  - 172.18.0.3:32379
✅ Deployment clustermesh-apiserver is ready
⌛ Waiting (0s) for clusters to be connected: 2 nodes are not ready
⌛ Waiting (11s) for clusters to be connected: 2 nodes are not ready
⌛ Waiting (24s) for clusters to be connected: 2 nodes are not ready
✅ All 2 nodes are connected to all clusters [min:1 / avg:1.0 / max:1]
🔌 Cluster Connections:
  - c2: 2/2 configured, 2/2 connected
🔀 Global services: [ min:0 / avg:0.0 / max:0 ]


cilium clustermesh status --context kind-c2 --wait
⚠️  Service type NodePort detected! Service may fail when nodes are removed from the cluster!
✅ Service "clustermesh-apiserver" of type "NodePort" found
✅ Cluster access information is available:
  - 172.18.0.4:32379
✅ Deployment clustermesh-apiserver is ready
✅ All 2 nodes are connected to all clusters [min:1 / avg:1.0 / max:1]
🔌 Cluster Connections:
  - c1: 2/2 configured, 2/2 connected
🔀 Global services: [ min:0 / avg:0.0 / max:0 ]
```

### Run connectivity test:

```yaml
cat deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rebel-base
spec:
  selector:
    matchLabels:
      name: rebel-base
  replicas: 2
  template:
    metadata:
      labels:
        name: rebel-base
    spec:
      containers:
      - name: rebel-base
        image: docker.io/nginx:1.15.8
        volumeMounts:
          - name: html
            mountPath: /usr/share/nginx/html/
        livenessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 1
        readinessProbe:
          httpGet:
            path: /
            port: 80
      volumes:
        - name: html
          configMap:
            name: rebel-base-response
            items:
              - key: message
                path: index.html
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: x-wing
spec:
  selector:
    matchLabels:
      name: x-wing
  replicas: 2
  template:
    metadata:
      labels:
        name: x-wing
    spec:
      containers:
      - name: x-wing-container
        image: docker.io/cilium/json-mock:1.2
        livenessProbe:
          exec:
            command:
            - curl
            - -sS
            - -o
            - /dev/null
            - localhost
        readinessProbe:
          exec:
            command:
            - curl
            - -sS
            - -o
            - /dev/null
            - localhost
```

```bash
kubectl --context kind-c1 apply -f deployment.yaml
kubectl --context kind-c2 apply -f deployment.yaml
```

```yaml
cat configmap_c1.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rebel-base-response
data:
  message: "{\"Cluster\": \"c1\", \"Planet\": \"N'Zoth\"}\n"

cat configmap_c2.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rebel-base-response
data:
  message: "{\"Cluster\": \"c2\", \"Planet\": \"Foran Tutha\"}\n"

kubectl --context kind-c1 apply -f configmap_c1.yaml
kubectl --context kind-c2 apply -f configmap_c2.yaml

cat service1.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: rebel-base
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    name: rebel-base
```

```bash
kubectl --context kind-c1 apply -f service1.yaml
kubectl --context kind-c2 apply -f service1.yaml
```

```bash
kubectl --context kind-c1 exec -ti deployment/x-wing -- curl rebel-base
{"Cluster": "c1", "Planet": "N'Zoth"}
kubectl --context kind-c1 exec -ti deployment/x-wing -- curl rebel-base
{"Cluster": "c1", "Planet": "N'Zoth"}

kubectl --context kind-c2 exec -ti deployment/x-wing -- curl rebel-base
{"Cluster": "c2", "Planet": "Foran Tutha"}
kubectl --context kind-c2 exec -ti deployment/x-wing -- curl rebel-base
{"Cluster": "c2", "Planet": "Foran Tutha"}
```

```yaml
cat service2.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: rebel-base
  annotations:
     io.cilium/global-service: "true"
     io.cilium/service-affinity: "local"
     io.cilium/shared-service: "false"
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    name: rebel-base
```

```bash
kubectl --context kind-c1 apply -f service2.yaml
kubectl --context kind-c2 apply -f service2.yaml
```

```bash
kubectl --context kind-c1 exec -ti deployment/x-wing -- curl rebel-base
{"Cluster": "c1", "Planet": "N'Zoth"}
kubectl --context kind-c1 exec -ti deployment/x-wing -- curl rebel-base
{"Cluster": "c2", "Planet": "Foran Tutha"}
```

