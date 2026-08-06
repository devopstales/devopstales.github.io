---
title: Change Certificates in Openshift
url: https://devopstales.github.io/kubernetes/openshift-cert/
date: 2019-04-17
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift
---


In this post I will show you how can you chnage certificate in Openshift.

<!--more-->

{{< content "/filedir/openshift.html" >}}

### Configure certs:
If you want to configure your Openshift cluster to use your own certificate you can do that wit this configuration.<br>
In my case the certificate files is MyCert.crt MyCert.key and the root CA is ccca.pem.

```
nano /ec/ansible/hosts
openshift_master_overwrite_named_certificates=true
openshift_hosted_router_certificate={"certfile": "/root/cert/MyCert.crt", "keyfile": "/root/cert/MyCert.key", "cafile": "/root/cert/ccca.pem"}
openshift_master_named_certificates=[{"names": ["master.openshit.devopstales.intra"],"certfile": "/root/cert/MyCert.crt", "keyfile": "/root/cert/MyCert.key", "cafile": "/root/cert/ccca.pem"}]

openshift_redeploy_openshift_ca=true
openshift_certificate_expiry_fail_on_warn=false

# registry
openshift_hosted_registry_routecertificates={"certfile": "/root/cert/MyCert.crt", "keyfile": "/root/cert/MyCert.key", "cafile": "/root/cert/ccca.pem"}
openshift_hosted_registry_routetermination=reencrypt
```

### Run the Installer
If your certificate is renewd you can cahge the certificate in the cluster with this playbooks.

```
oc get csr | grep Pending | awk '{print $1}' | xargs oc adm certificate approve

ansible-playbook -i hosts /usr/share/ansible/openshift-ansible/playbooks/redeploy-certificates.yml

ansible-playbook -i hosts /usr/share/ansible/openshift-ansible/playbooks/openshift-master/redeploy-openshift-ca.yml
ansible-playbook -i hosts /usr/share/ansible/openshift-ansible/playbooks/openshift-etcd/redeploy-ca.yml
```

