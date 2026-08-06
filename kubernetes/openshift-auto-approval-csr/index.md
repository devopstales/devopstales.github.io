---
title: How to Enable Auto Approval of CSR in Openshift v3.11
url: https://devopstales.github.io/kubernetes/openshift-auto-approval-csr/
date: 2020-05-27
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift
---



Nodes certificates are not Completely redeployed through playbook but through a different mechanism.

<!--more-->

{{< content "/filedir/openshift.html" >}}

SSL Certificates will be valid for the period of 1 year and at 85% of the certificate the node will trigger a CSR that would have to be approved for the certificate to be redeployed.

The Only certificates that are renewed/redeployed through CSR’s mechanism are the kubelet/nodes certificates. Any other certificates e.g, router, master, api certs, etcd, docker-registry, etc are still redeployed through the usual playbooks.

If triggered CSR is not approved either manually or in automated way then after one year all nodes will go to NotReady State.

### Check and approve csr's manually
```
oc get csr
oc describe csr <csr_name>
oc adm certificate <approve csr_name>

oc get csr -o name | xargs oc adm certificate approve
```

### Approve csr's automaticle

At install time you can add this option to your ansible hosts fiel:
```
openshift_master_bootstrap_auto_approve=true
```

If you installed the cluster and want to change this option run this playbook:
```
ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/openshift-master/enable_bootstrap.yml \
-e openshift_master_bootstrap_auto_approve=true
```

