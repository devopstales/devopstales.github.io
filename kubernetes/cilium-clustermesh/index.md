---
title: Kubernetes Multicluster with Cilium Cluster Mesh
url: https://devopstales.github.io/kubernetes/cilium-clustermesh/
date: 2021-10-06
keywords: Kubernetes, k8s, Load Balancer, BGP, kubernetes deployment, kubernetes ingress, Cilium, Cluster Mesh, Multi-Cluster, clustermesh
---


In this tutorial I will show you how to install Cilium on multiple Kubernetes clusters and connect those clusters with Cluster Mesh.

<!--more-->

### What is Cluster Mesh

Cluster mesh extends the networking datapath across multiple clusters. It allows endpoints in all connected clusters to communicate while providing full policy enforcement. Load-balancing is available via Kubernetes annotations.

### The infrastructure

```bash
k3s-cl1:
ip: 172.17.11.11

k3s-cl2:
ip: 172.17.11.12
etcd
kube-vip

k3s-cl3:
ip: 172.17.11.13
etcd
kube-vip
```

## Installing k3s with k3sup

```bash
ssh-copy-id vagrant@172.17.11.11
ssh-copy-id vagrant@172.17.11.12
ssh-copy-id vagrant@172.17.11.13
```

```bash
tmux-cssh vagrant@172.17.11.11 vagrant@172.17.11.12 vagrant@172.17.11.13
sudo su -
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
k3sup --help
```

### Bootstrap cl1

```bash
ssh vagrant@172.17.11.11
sudo su -

k3sup install \
  --local \
  --ip=172.17.11.11 \
  --cluster \
  --k3s-channel=stable \
  --k3s-extra-args "--flannel-backend=none --cluster-cidr=10.11.0.0/16 --disable-network-policy --no-deploy=traefik --no-deploy=servicelb --node-ip=172.17.11.11" \
  --merge \
  --local-path $HOME/.kube/config \
  --context=k3scl1

exit
exit
```

### Bootstrap cl2

```bash
ssh vagrant@172.17.11.12
sudo su -

k3sup install \
  --local \
  --ip=172.17.11.12 \
  --cluster \
  --k3s-channel=stable \
  --k3s-extra-args "--flannel-backend=none --cluster-cidr=10.12.0.0/16 --disable-network-policy --no-deploy=traefik --no-deploy=servicelb --node-ip=172.17.11.12" \
  --merge \
  --local-path $HOME/.kube/config \
  --context=k3scl2

exit
exit
```

### Bootstrap cl3

```bash
ssh vagrant@172.17.11.13
sudo su -

k3sup install \
  --local \
  --ip=172.17.11.13 \
  --cluster \
  --k3s-channel=stable \
  --k3s-extra-args "--flannel-backend=none --cluster-cidr=10.13.0.0/16 --disable-network-policy --no-deploy=traefik --no-deploy=servicelb --node-ip=172.17.11.13" \
  --merge \
  --local-path $HOME/.kube/config \
  --context=k3scl3

exit
exit
```

## Deploy cilium

Each Kubernetes cluster maintains its own etcd cluster which contains the state of that cluster's cilium. State from multiple clusters is never mixed in etcd itself. Each cluster exposes its own etcd via a set of etcd proxies. In this demo I will use NodePort service. Cilium agents running in other clusters connect to the etcd proxies to watch for changes and replicate the multi-cluster relevant state into their own cluster. Use of etcd proxies ensures scalability of etcd watchers. Access is protected with TLS certificates.

![Cilium clustermesh](/img/include/clustermesh-arch.webp)

```bash
ssh vagrant@172.17.11.11
sudo su -

helm repo add cilium https://helm.cilium.io/
helm repo update
helm upgrade --install cilium cilium/cilium \
--set operator.replicas=1 \
--set cluster.id=1 \
--set cluster.name=k3scl1 \
--set tunnel=vxlan \
--set kubeProxyReplacement=strict \
--set containerRuntime.integration=containerd \
--set etcd.enabled=true \
--set etcd.managed=true \
--set k8sServiceHost=10.0.2.15 \
--set k8sServicePort=6443 \
-n kube-system

cat <<EOF | kubectl -n kube-system apply -f -
apiVersion: v1
kind: Service
metadata:
  name: cilium-etcd-external
spec:
  type: NodePort
  ports:
  - port: 2379
  selector:
    app: etcd
    etcd_cluster: cilium-etcd
    io.cilium/app: etcd-operator
EOF

exit
exit
```

```bash
ssh vagrant@172.17.11.12
sudo su -

helm repo add cilium https://helm.cilium.io/
helm repo update
helm upgrade --install cilium cilium/cilium \
--set operator.replicas=1 \
--set cluster.id=2 \
--set cluster.name=k3scl2 \
--set tunnel=vxlan \
--set containerRuntime.integration=containerd \
--set etcd.enabled=true \
--set etcd.managed=true \
--set k8sServiceHost=10.0.2.15 \
--set k8sServicePort=6443 \
-n kube-system

cat <<EOF | kubectl -n kube-system apply -f -
apiVersion: v1
kind: Service
metadata:
  name: cilium-etcd-external
spec:
  type: NodePort
  ports:
  - port: 2379
  selector:
    app: etcd
    etcd_cluster: cilium-etcd
    io.cilium/app: etcd-operator
EOF

exit
exit
```

```bash
ssh vagrant@172.17.11.13
sudo su -

helm repo add cilium https://helm.cilium.io/
helm repo update
helm upgrade --install cilium cilium/cilium \
--set operator.replicas=1 \
--set cluster.id=3 \
--set cluster.name=k3scl3 \
--set tunnel=vxlan \
--set containerRuntime.integration=containerd \
--set etcd.enabled=true \
--set etcd.managed=true \
--set k8sServiceHost=10.0.2.15 \
--set k8sServicePort=6443 \
-n kube-system

cat <<EOF | kubectl -n kube-system apply -f -
apiVersion: v1
kind: Service
metadata:
  name: cilium-etcd-external
spec:
  type: NodePort
  ports:
  - port: 2379
  selector:
    app: etcd
    etcd_cluster: cilium-etcd
    io.cilium/app: etcd-operator
EOF

exit
exit
```

### Configure cluster mesh

```bash
ssh vagrant@172.17.11.11
sudo su -

cd /tmp
git clone https://github.com/cilium/clustermesh-tools.git
cd clustermesh-tools

./extract-etcd-secrets.sh 
Derived cluster-name k3scl1 from present ConfigMap
====================================================
 WARNING: The directory config contains private keys.
          Delete after use.
====================================================

exit
exit
```

```bash
ssh vagrant@172.17.11.12
sudo su -

cd /tmp
git clone https://github.com/cilium/clustermesh-tools.git
cd clustermesh-tools

./extract-etcd-secrets.sh 
Derived cluster-name k3scl2 from present ConfigMap
====================================================
 WARNING: The directory config contains private keys.
          Delete after use.
====================================================

scp -r config/ 172.17.11.11:/tmp/clustermesh-tools/

exit
exit
```

```bash
ssh vagrant@172.17.11.13
sudo su -

cd /tmp
git clone https://github.com/cilium/clustermesh-tools.git
cd clustermesh-tools

./extract-etcd-secrets.sh 
Derived cluster-name k3scl3 from present ConfigMap
====================================================
 WARNING: The directory config contains private keys.
          Delete after use.
====================================================

scp -r config/ 172.17.11.11:/tmp/clustermesh-tools/

exit
exit
```

```bash
ssh vagrant@172.17.11.11
sudo su -

cd /tmp/clustermesh-tools/
./generate-secret-yaml.sh > clustermesh.yaml
./generate-name-mapping.sh > ds.patch

scp clustermesh.yaml ds.patch 172.17.11.12:
scp clustermesh.yaml ds.patch 172.17.11.13:

kubectl -n kube-system patch ds cilium -p "$(cat ds.patch)"
kubectl -n kube-system apply -f clustermesh.yaml
kubectl -n kube-system delete pod -l k8s-app=cilium
kubectl -n kube-system delete pod -l name=cilium-operator

ssh 172.17.11.12 '
  kubectl -n kube-system patch ds cilium -p "$(cat ds.patch)"
  kubectl -n kube-system apply -f clustermesh.yaml
  kubectl -n kube-system delete pod -l k8s-app=cilium
  kubectl -n kube-system delete pod -l name=cilium-operator
'

ssh 172.17.11.13 '
  kubectl -n kube-system patch ds cilium -p "$(cat ds.patch)"
  kubectl -n kube-system apply -f clustermesh.yaml
  kubectl -n kube-system delete pod -l k8s-app=cilium
  kubectl -n kube-system delete pod -l name=cilium-operator
'
```

Verify the cluster mesh by dumping the node list from any cilium. It should show all nodes in both the clusters.

```bash
kubectl get po -l k8s-app=cilium
NAME           READY   STATUS    RESTARTS   AGE
cilium-6z8zf   1/1     Running   0          3m54s

kubectl -n kube-system exec -ti cilium-6z8zf -- cilium node list
 
Defaulted container "cilium-agent" out of: cilium-agent, mount-cgroup (init), clean-cilium-state (init)
Name           IPv4 Address   Endpoint CIDR   IPv6 Address   Endpoint CIDR
k3scl1/k3s01   172.17.11.11   10.0.0.0/24                    
k3scl2/k3s02   172.17.11.12   10.0.0.0/24                    
k3scl3/k3s03   172.17.11.13   10.0.0.0/24      
```

### Test the connection

```yaml
ssh vagrant@172.17.11.11
sudo su -

kubectl create ns test
kubens test

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1
        ports:
        - name: http
          containerPort: 80
EOF
```

The service discovery of Cilium's multi-cluster model is built using standard Kubernetes services and designed to be completely transparent to existing Kubernetes application deployments:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-global
  namespace: test
  annotations:
    io.cilium/global-service: "true"
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
EOF
```

Cilium monitors Kubernetes services and endpoints and watches for services with an annotation `io.cilium/global-service: "true"`. For such services, all services with identical name and namespace information are automatically merged together and form a global service that is available across clusters. 

### Test cl1

After you deploy the global service to both of the clusters, you can test the connection:

```bash
kubectl -n kube-system exec -ti cilium-6z8zf -- cilium service list 
Defaulted container "cilium-agent" out of: cilium-agent, mount-cgroup (init), clean-cilium-state (init)
ID   Frontend             Service Type   Backend                  
...
5    10.43.200.118:80     ClusterIP      1 => 10.0.0.145:80       
                                         2 => 10.0.0.69:80        
...

kubectl -n kube-system exec -ti cilium-6z8zf -- cilium bpf lb list 
Defaulted container "cilium-agent" out of: cilium-agent, mount-cgroup (init), clean-cilium-state (init)
SERVICE ADDRESS      BACKEND ADDRESS
...
10.43.200.118:80     10.0.0.69:80 (5)                          
                     10.0.0.145:80 (5)                         
...
```

```bash
kubectl run -it --rm --image=tianon/network-toolbox debian
curl nginx-global
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Test on cl3:

```bash
ssh 172.17.11.12

kubectl create ns test
kubens test

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-global
  namespace: test
  annotations:
    io.cilium/global-service: "true"
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
EOF

kubectl -n kube-system exec -ti cilium-8wzrc -- cilium service list
Defaulted container "cilium-agent" out of: cilium-agent, mount-cgroup (init), clean-cilium-state (init)
ID   Frontend            Service Type   Backend                  
...
8    10.43.189.53:80     ClusterIP      1 => 10.0.0.69:80        
                                        2 => 10.0.0.145:80
...

kubectl -n kube-system exec -ti cilium-8wzrc -- cilium bpf lb list
Defaulted container "cilium-agent" out of: cilium-agent, mount-cgroup (init), clean-cilium-state (init)
SERVICE ADDRESS     BACKEND ADDRESS
...
10.43.189.53:80     0.0.0.0:0 (8) [ClusterIP, non-routable]   
                    10.0.0.69:80 (8)                          
                    10.0.0.145:80 (8)                    
...

kubectl run -it --rm --image=tianon/network-toolbox debian
curl nginx-global
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

### Multi-cluster network policy

It is possible to establish policies that apply to pod in particular clusters only. The cluster name is represented as a label on each pod by Cilium which allows to match on the cluster name in both the `endpointSelector` as well as the `matchLabels` for `toEndpoints` and fromEndpoints constructs:

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "allow-cross-cluster"
  description: "Allow x-wing in cluster1 to contact rebel-base in cluster2"
spec:
  endpointSelector:
    matchLabels:
      name: x-wing
      io.cilium.k8s.policy.cluster: cluster1
  egress:
  - toEndpoints:
    - matchLabels:
        name: rebel-base
        io.cilium.k8s.policy.cluster: cluster2
```

The above example policy will allow `x-wing` in cluster1 to talk to `rebel-base` in cluster2. X-wings won't be able to talk to rebel bases in the local cluster unless additional policies exist that whitelist the communication.

