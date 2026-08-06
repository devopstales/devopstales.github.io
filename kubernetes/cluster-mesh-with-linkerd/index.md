---
title: Multicluster Kubernetes with Linkerd Cluster Mesh
url: https://devopstales.github.io/kubernetes/cluster-mesh-with-linkerd/
date: 2023-08-06
keywords: Kubernetes, k8s, Load Balancer, kubernetes deployment, kubernetes ingress, linkerd, Cluster Mesh, Multi-Cluster, clustermesh
---


In this tutorial I will show you how to install linkerd on multiple Kubernetes clusters and connect those clusters with Cluster Mesh.

<!--more-->

### Bootstrap kind clustes with loadbalancer

```yaml
nano kind-c1-config.yaml
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
networking:
  apiServerAddress: 192.168.0.15 # PUT YOUR IP ADDRESSS OF YOUR MACHINE HERE!
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
  apiServerAddress: 192.168.0.15 # PUT YOUR IP ADDRESSS OF YOUR MACHINE HERE!
  podSubnet: "10.2.0.0/16"
  serviceSubnet: "10.3.0.0/16"
```

```bash
docker network inspect -f '{{.IPAM.Config}}' kind
[{172.18.0.0/16  172.18.0.1 map[]} {fc00:f853:ccd:e793::/64  fc00:f853:ccd:e793::1 map[]}]
```

```yaml
nano c1-address-pool.yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.150-172.18.255.199
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
```

```yaml
nano c2-address-pool.yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.150-172.18.255.199
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
```



```bash
kind create cluster --name c1 --config kind-c1-config.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
kubectl apply -f c1-address-pool.yaml

kind create cluster --name c2 --config kind-c2-config.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
kubectl apply -f c2-address-pool.yaml
```


### Generate Certificates for Linkerd
```bash
brew install step
brew install linkerd
curl -sL https://linkerd.github.io/linkerd-smi/install | sh

step certificate create root.linkerd.cluster.local root.crt root.key \
  --profile root-ca --no-password --insecure

step certificate create identity.linkerd.cluster.local issuer.crt issuer.key \
  --profile intermediate-ca --not-after 8760h --no-password --insecure \
  --ca root.crt --ca-key root.key
```

```bash
# first, install the Linkerd CRDs in both clusters
linkerd install --crds | tee \
    >(kubectl --context=kind-c1 apply -f -) \
    >(kubectl --context=kind-c2 apply -f -)

# then install the Linkerd control plane in both clusters
linkerd install \
  --identity-trust-anchors-file root.crt \
  --identity-issuer-certificate-file issuer.crt \
  --identity-issuer-key-file issuer.key \
  | tee \
    >(kubectl --context=kind-c1 apply -f -) \
    >(kubectl --context=kind-c2 apply -f -)
```

```bash
# And then Linkerd Viz:

for ctx in kind-c1 kind-c2; do
  linkerd --context=${ctx} viz install | \
    kubectl --context=${ctx} apply -f - || break
done

# And SMI plugin

for ctx in kind-c1 kind-c2; do
    echo "install smi ${ctx}";        
    linkerd smi install --context=${ctx}  | kubectl apply -f - --context=${ctx};
done

for ctx in kind-c1 kind-c2; do
  echo "Checking cluster: ${ctx} ........."
  linkerd --context=${ctx} check || break
  echo "-------------"
done
```

To install the multicluster components on both cluster, you can run:

```bash
for ctx in kind-c1 kind-c2; do
  echo "Installing on cluster: ${ctx} ........."
  linkerd --context=${ctx} multicluster install | \
    kubectl --context=${ctx} apply -f - || break
  echo "-------------"
done
```

```bash
for ctx in kind-c1 kind-c2; do
  echo "Checking gateway on cluster: ${ctx} ........."
  kubectl --context=${ctx} -n linkerd-multicluster \
    rollout status deploy/linkerd-gateway || break
  echo "-------------"
done

for ctx in kind-c1 kind-c2; do
  printf "Checking cluster: ${ctx} ........."
  while [ "$(kubectl --context=${ctx} -n linkerd-multicluster get service linkerd-gateway -o 'custom-columns=:.status.loadBalancer.ingress[0].ip' --no-headers)" = "<none>" ]; do
      printf '.'
      sleep 1
  done
  echo "`kubectl --context=${ctx} -n linkerd-multicluster get service linkerd-gateway -o 'custom-columns=:.status.loadBalancer.ingress[0].ip' --no-headers`"
  printf "\n"
done
```

### Link the clusters

```bash
for ctx in kind-c1 kind-c2; do
  linkerd --context=${ctx} multicluster check -o short
done
```

```bash
linkerd --context kind-c1 multicluster link --cluster-name kind-c1 | kubectl --context=kind-c2 apply -f -
linkerd --context kind-c2 multicluster link --cluster-name kind-c2 | kubectl --context=kind-c1 apply -f -
```

```bash
for ctx in kind-c1 kind-c2; do
  linkerd --context=${ctx} multicluster check -o short
done

Status check results are √
Status check results are √
```

```bash
for ctx in kind-c1 kind-c2; do
  linkerd --context=${ctx} multicluster gateways
done

CLUSTER  ALIVE    NUM_SVC      LATENCY  
kind-c2  True           0          1ms  
CLUSTER  ALIVE    NUM_SVC      LATENCY  
kind-c1  True           0          4ms 
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
      annotations:
        linkerd.io/inject: enabled
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
      annotations:
        linkerd.io/inject: enabled
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
```

```bash
kubectl --context kind-c1 apply -f configmap_c1.yaml
kubectl --context kind-c2 apply -f configmap_c2.yaml
```

```yaml
cat service.yaml
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
kubectl --context kind-c1 apply -f service.yaml
kubectl --context kind-c2 apply -f service.yaml
```

```bash
kubectl --context kind-c1 exec -ti deployment/x-wing -c x-wing-container -- curl rebel-base
{"Cluster": "c1", "Planet": "N'Zoth"}
kubectl --context kind-c1 exec -ti deployment/x-wing -c x-wing-container -- curl rebel-base
{"Cluster": "c1", "Planet": "N'Zoth"}

kubectl --context kind-c2 exec -ti deployment/x-wing -c x-wing-container -- curl rebel-base
{"Cluster": "c2", "Planet": "Foran Tutha"}
kubectl --context kind-c2 exec -ti deployment/x-wing -c x-wing-container -- curl rebel-base
{"Cluster": "c2", "Planet": "Foran Tutha"}

```

### Expose the service

```bash
kubectl --context kind-c1 label svc -n default rebel-base mirror.linkerd.io/exported=true
```

```bash
kubectl --context kind-c2 get svc -n default
NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes           ClusterIP   10.1.0.1      <none>        443/TCP   34m
rebel-base           ClusterIP   10.1.52.213   <none>        80/TCP    2m13s
rebel-base-kind-c1   ClusterIP   10.1.140.47   <none>        80/TCP    30s
```

```yaml
nano TrafficSplit.yaml
---
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: rebel-base
  namespace: default
spec:
  service: rebel-base
  backends:
  - service: rebel-base
    weight: 50
  - service: rebel-base-kind-c1
    weight: 50
```

```bash
kubectl --context kind-c2 apply -f TrafficSplit.yaml
```

```bash
kubectl --context kind-c2 exec -ti deployment/x-wing -c x-wing-container -- curl rebel-base
{"Cluster": "c2", "Planet": "Foran Tutha"}

kubectl --context kind-c2 exec -ti deployment/x-wing -c x-wing-container -- curl rebel-base
{"Cluster": "c1", "Planet": "N'Zoth"}
```

```bash
kubectl --context kind-c1 exec -ti deployment/x-wing -c x-wing-container -- curl rebel-base
{"Cluster": "c1", "Planet": "N'Zoth"}
kubectl --context kind-c1 exec -ti deployment/x-wing -c x-wing-container -- curl rebel-base
{"Cluster": "c1", "Planet": "N'Zoth"}```
