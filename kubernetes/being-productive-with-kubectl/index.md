---
title: Being Productive with K8S
url: https://devopstales.github.io/kubernetes/being-productive-with-kubectl/
date: 2020-10-09
keywords: Kubernetes, K8s, Productive
---


In this post I will show you my productivity tips with kubectl.
<!--more-->

### Bash aliases for kubectl
I have above aliases setup in the `~/.bashrc` file.

```
alias k=kubectl
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kdelf='kubectl delete -f'
alias kaf='kubectl apply -f'
alias keti='kubectl exec -ti'
alias kgds='kubectl get DaemonSet'
alias kgdsy='kubectl get DaemonSet -oyaml'
alias kdds='kubectl describe DaemonSet'
alias kdelds='kubectl delete DaemonSet'
alias kgp='kubectl get pods'
alias kgpy='kubectl get pods -oyaml'
alias kgpa='kubectl get pods --all-namespaces'
alias kgpw='kgp --watch'
alias kgpwide='kgp -o wide'
alias kdp='kubectl describe pods'
alias kdelp='kubectl delete pods'
# get pod by label: kgpl "app=myapp" -n myns
alias kgpl='kgp -l'
# get pod by namespace: kgpn kube-system"
alias kgpn='kgp -n'
alias kgs='kubectl get svc'
alias kgsa='kubectl get svc --all-namespaces'
alias kgsw='kgs --watch'
alias kgswide='kgs -o wide'
alias kes='kubectl edit svc'
alias kds='kubectl describe svc'
alias kdels='kubectl delete svc'
alias kgi='kubectl get ingress'
alias kgia='kubectl get ingress --all-namespaces'
alias kei='kubectl edit ingress'
alias kdi='kubectl describe ingress'
alias kdeli='kubectl delete ingress'
alias kgns='kubectl get namespaces'
alias kens='kubectl edit namespace'
alias kdns='kubectl describe namespace'
alias kdelns='kubectl delete namespace'
alias kcn='kubectl config set-context $(kubectl config current-context) --namespace'
alias kgcm='kubectl get configmaps'
alias kgcmy='kubectl get configmaps -oyaml'
alias kgcma='kubectl get configmaps --all-namespaces'
alias kecm='kubectl edit configmap'
alias kdcm='kubectl describe configmap'
alias kdelcm='kubectl delete configmap'
alias kgsec='kubectl get secret'
alias kgseca='kubectl get secret --all-namespaces'
alias kdsec='kubectl describe secret'
alias kdelsec='kubectl delete secret'
alias kgss='kubectl get statefulset'
alias kgssa='kubectl get statefulset --all-namespaces'
alias kgssw='kgss --watch'
alias kgsswide='kgss -o wide'
alias kess='kubectl edit statefulset'
alias kdss='kubectl describe statefulset'
alias kdelss='kubectl delete statefulset'
alias ksss='kubectl scale statefulset'
alias krsss='kubectl rollout status statefulset'
alias kga='kubectl get all'
alias kgaa='kubectl get all --all-namespaces'
alias kl='kubectl logs'
alias kl1h='kubectl logs --since 1h'
alias kl1m='kubectl logs --since 1m'
alias kl1s='kubectl logs --since 1s'
alias klf='kubectl logs -f'
alias klf1h='kubectl logs --since 1h -f'
alias klf1m='kubectl logs --since 1m -f'
alias klf1s='kubectl logs --since 1s -f'
alias kcp='kubectl cp'
alias kgno='kubectl get nodes'
alias keno='kubectl edit node'
alias kdno='kubectl describe node'
alias kdelno='kubectl delete node'
alias kgpvc='kubectl get pvc'
alias kgpvca='kubectl get pvc --all-namespaces'
alias kgpvcw='kgpvc --watch'
alias kepvc='kubectl edit pvc'
alias kdpvc='kubectl describe pvc'
alias kdelpvc='kubectl delete pvc'
alias kgpv='kubectl get pv'
alias kgsc='kubectl get storageclass'
# custom resource
alias kgir='kubectl get IngressRoute'
alias kgira='kubectl get IngressRoute --all-namespaces'
alias keir='kubectl edit IngressRoute'
alias kdir='kubectl describe IngressRoute'
alias kdelir='kubectl delete IngressRoute'
```

### kubens kubectx
```
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
echo "PATH=$PATH:/usr/local/bin/" >> /etc/profile
```

