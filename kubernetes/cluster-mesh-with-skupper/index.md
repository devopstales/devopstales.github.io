---
title: Multicluster Kubernetes with Skupper Cluster Mesh
url: https://devopstales.github.io/kubernetes/cluster-mesh-with-skupper/
date: 2023-08-12
keywords: Kubernetes, k8s, Load Balancer, kubernetes deployment, kubernetes ingress, skupper, Cluster Mesh, Multi-Cluster, clustermesh
---


In this tutorial I will show you how to install skupper on multiple Kubernetes clusters and connect those clusters with Cluster Mesh.

<!--more-->

```bash
wget https://github.com/skupperproject/skupper/releases/download/1.4.2/skupper-cli-1.4.2-linux-amd64.tgz
tar -xzf skupper-cli-1.4.2-linux-amd64.tgz

mkdir -p $HOME/bin
mv skupper $HOME/bin
export PATH=$PATH:$HOME/bin
```

```bash
kind create cluster --name c1
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
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


```bash
kind create cluster --name c2
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
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

### Install Skuuper

```bash
kubectx kind-c1

kubectl create ns interconnect
kubens interconnect

skupper init --enable-cluster-permissions --enable-console --enable-flow-collector
```

```bash
kubectx kind-c2

kubectl create ns interconnect
kubens interconnect

skupper init --enable-cluster-permissions
```

```bash
skupper token create c2-token.yaml

kubectx kind-c1

skupper link create c2-token.yaml
```

### Test deploy

```yaml
nano deploy.yaml
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

```yaml
nano configmap_c1.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rebel-base-response
data:
  message: "{\"Cluster\": \"c1\", \"Planet\": \"N'Zoth\"}\n"

```

```yaml
nano configmap_c2.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rebel-base-response
data:
  message: "{\"Cluster\": \"c2\", \"Planet\": \"Foran Tutha\"}\n"
```

```bash
kubectx kind-c1
kubens interconnect
kubectl apply -f deploy.yaml
kubectl apply -f configmap_c1.yaml
kubectl apply -f service.yaml

kubectx kind-c2
kubens interconnect
kubectl apply -f deploy.yaml
kubectl apply -f configmap_c2.yaml
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

```bash
skupper expose deployment/rebel-base \
  --port 80 -c kind-c2

skupper expose deployment/rebel-base \
  --port 80 -c kind-c1
```

```yaml
kgs rebel-base -o yaml           
apiVersion: v1
kind: Service
metadata:
  annotations:
    internal.skupper.io/originalAssignedPort: 80:1024
    internal.skupper.io/originalSelector: name=rebel-base
    internal.skupper.io/originalTargetPort: 80:80
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"rebel-base","namespace":"interconnect"},"spec":{"ports":[{"port":80}],"selector":{"name":"rebel-base"},"type":"ClusterIP"}}
  creationTimestamp: "2023-08-25T07:48:10Z"
  name: rebel-base
  namespace: interconnect
  resourceVersion: "4874"
  uid: 584f78e7-ca4f-4b7e-bb22-ff57df56552b
spec:
  clusterIP: 10.96.209.144
  clusterIPs:
  - 10.96.209.144
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 1024
  selector:
    application: skupper-router
    skupper.io/component: router
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```


```bash
kubectl --context kind-c2 exec -ti deployment/x-wing -- curl rebel-base
{"Cluster": "c2", "Planet": "Foran Tutha"}

kubectl --context kind-c2 exec -ti deployment/x-wing -- curl rebel-base
{"Cluster": "c1", "Planet": "N'Zoth"}
```


### Admin Console

```bash
kubectl get service skupper                                                                        
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                        AGE
skupper   LoadBalancer   10.96.135.212   172.18.255.201   8010:31238/TCP,8080:32514/TCP,8081:32366/TCP   32m
```

```
kubectl get secret skupper-console-users -o json | jq -r .data.admin | base64 -d
xfGRpW09Qu
```

Open `https://172.18.255.201:8010` in your browser and login with `admin` as user and `xfGRpW09Qu` as password.

