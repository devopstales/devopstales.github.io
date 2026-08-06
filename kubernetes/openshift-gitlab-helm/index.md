---
title: Installing GitLab on OpenShift
url: https://devopstales.github.io/kubernetes/openshift-gitlab-helm/
date: 2019-11-18
keywords: ansible, openshift 3.11, openshift okd, openshift tutorial, openshift container platform, rad hat openshift, helm 3, gitlab org chart, gitlab cicd
---


I  had to install Gitlab to Openshift recently. Turned out getting GitLab up and running on OpenShift is not so easy.

<!--more-->

### Create new project
```
oc new-project gitlab-devopstales.intra
```

### Deploy helm
```
nano helm-namespace-account.yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: tiller-gitlab-devopstales.intra
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller-gitlab-devopstales.intra
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller-gitlab-devopstales.intra
    namespace: kube-system
```

Now set up Helm, install the Tiller plugin and add the GitLab repository.

```
oc apply -f helm-namespace-account.yaml
oc get sa

helm init --service-account tiller-gitlab-devopstales.intra --tiller-namespace gitlab-mydomain-intra
oc get po -n kube-system

export TILLER_NAMESPACE=kube-system
echo $TILLER_NAMESPACE
helm version
```

## Get helmchart
```
helm repo add gitlab https://charts.gitlab.io/
helm repo update

oc adm policy add-scc-to-user anyuid -z default -n gitlab-devopstales.intra
oc adm policy add-scc-to-user anyuid -z gitlab-runner -n gitlab-devopstales.intra

# gitlab-tst is the name of the helm deployment
oc adm policy add-scc-to-user anyuid -z gitlab-tst-shared-secrets
oc adm policy add-scc-to-user anyuid -z gitlab-tst-gitlab-runner
oc adm policy add-scc-to-user anyuid -z gitlab-tst-prometheus-server
oc adm policy add-scc-to-user anyuid -z default
```

### Create chart values
```
nano gitlab-values.yml
certmanager:
  install: false
global:
  appConfig:
    enableUsagePing: true
    enableImpersonation: true
    defaultCanCreateGroup: true
    usernameChangingEnabled: true
    issueClosingPattern:
    defaultTheme:
    defaultProjectsFeatures:
      issues: true
      mergeRequests: true
      wiki: true
      snippets: true
      builds: true
      containerRegistry: true
    ldap:
      servers:
        main:
          base: dc=mydomain,dc=intra
          user_filter: (&(objectClass=user)(memberof=cn=Users,dc=mydomain,dc=intra))
          bind_dn: Administrator@devopstales.intra
          host: 192.168.10.4
          label: devopstales.intra
          password:
            key: password
            secret: gitlab-ldap-secret
          port: 636
          encryption: simple_tls
          uid: sAMAccountName
          active_directory: true
          verify_certificates: false
          allow_username_or_email_login: true
    omniauth:
      enabled: true
      blockAutoCreatedUsers: false
      allowSingleSignOn: ['oauth2_generic']
      providers:
        - secret: gitlab-sso
          key: provider
    backups:
      bucket: gitlab-devopstales.intra
      tmpBucket: gitlab-devopstales.intra
      objectStorage:
        backend: s3
    lfs:
      bucket: gitlab-devopstales.intra
      connection:
        secret: ceph-storage
        key: gitlab
    artifacts:
      bucket: gitlab-devopstales.intra
      connection:
        secret: ceph-storage
        key: gitlab
    uploads:
      bucket: gitlab-devopstales.intra
      connection:
        secret: ceph-storage
        key: gitlab
    packages:
      bucket: gitlab-devopstales.intra
      connection:
        secret: ceph-storage
        key: gitlab
    externalDiffs:
      bucket: gitlab-devopstales.intra
      connection:
        secret: ceph-storage
        key: gitlab
    pseudonymizer:
      bucket: gitlab-devopstales.intra
      connection:
        secret: ceph-storage
        key: gitlab
  edition: ce
  email:
    from: gitlab@devopstales.intra
  hosts:
    domain: devopstales.intra
    externalIP: gitlab.devopstales.intra
    gitlab:
      name: gitlab.devopstales.intra
      https: false
    registry:
      name: gitlab-registry.devopstales.intra
      https: false
  ingress:
    enabled: false
    configureCertmanager: false
    tls:
      secretName: gitlab-certs
  smtp:
    address: mail.active.hu
    authentication: ""
    domain: devopstales.intra
    enabled: true
    port: 25
  gitlab-exporter:
    enabled: false
  registry:
    bucket: gitlab-registry
  minio:
    enabled: false
nginx-ingress:
  enabled: false
gitlab-runner:
  rbac:
    create: true
registry:
  enabled: true
  storage:
    secret: ceph-storage
    key: registry
  image:
    repository: docker.io/registry
    tag: 2.6.0
gitlab:
  task-runner:
    backups:
      objectStorage:
        config:
          secret: storage-config
          key: config
```

### Create secrets for deployment
```
nano ceph.gitlab-data.yaml
provider: AWS
region: default
aws_access_key_id: W3MNDO373H6LQUNCG4SG
aws_secret_access_key: vVFEWx3hqbcrGJyaZVie9YoFG6rPoRYmqnDzRwrn
endpoint: "https://s3.devopstales.intra"
enable_signature_v4_streaming: false

# admin jog kell a cephez
nano ceph.gitlab-registry.yaml
cache:
  blobdescriptor: inmemory
s3:
  region: default
  bucket: gitlab-registry
  accesskey: PZIOIH63CENHPG15XY42
  secretkey: K6K1lWO7Jtyp5rZiCwj77JC5BFMEAZ4a2PAkg9fB
  regionendpoint: https://s3.devopstales.intra
  rootdirectory: /
  secure: true
  v4auth: false
  encrypt: false
  chunksize: 5242880
redirect:
  disable: true
```

```
nano ceph.backup.config
[default]
access_key = W3MNDO373H8LQUNCJ8QV
access_token = vVFEWx8hqbcrGJyaZVie8YoER8rPoRYmqnDzRwrn
host_base = s3.devopstales.intra
host_bucket = %(bucket)s.s3.devopstales.intra
bucket_location = US
use_https = True
check_ssl_certificate = False
```

```
nano keycloak.sso.yaml
name: 'oauth2_generic'
label: 'mydomain'
app_id: 'gitlab'
app_secret: 'f2514bd4-92e4-40fa-bec4-382838db25f0'
args:
  client_options:
    site: 'https://sso.devopstales.intra'
    user_info_url: '/auth/realms/mydomain/protocol/openid-connect/userinfo'
    authorize_url: '/auth/realms/mydomain/protocol/openid-connect/auth'
    token_url: '/auth/realms/mydomain/protocol/openid-connect/token'
  user_response_structure:
    attributes:
      email: 'email'
      first_name: 'given_name'
      last_name: 'family_name'
      name: 'name'
      nickname: 'preferred_username'
    id_path: 'preferred_username'
```

### Deploy secrets
```
# https ssl cert
oc create secret tls gitlab-certs --cert=tls.crt --key=tls.key

oc create secret generic storage-config --from-file=config=ceph.backup.config

oc create secret generic ceph-storage --from-file=registry=ceph.gitlab-registry.yaml --from-file=gitlab=ceph.gitlab-data.yaml

oc create secret generic gitlab-sso --from-file=provider=keycloak.sso.yaml
oc create secret generic gitlab-ldap-secret --from-literal=password=
```


### Deploy application with helm
```
helm upgrade --install -f gitlab-values.yml gitlab-tst gitlab/gitlab --debug --dry-run
helm upgrade --install -f gitlab-values.yml gitlab-tst gitlab/gitlab --timeout 600
helm upgrade -f gitlab-values.yml gitlab-tst gitlab/gitlab --timeout 600

# https://docs.gitlab.com/charts/installation/version_mappings.html
helm upgrade -f gitlab-values.yml gitlab-tst gitlab/gitlab --version 2.3.5 --timeout 600

# gitlab-tst
oc get secret gitlab-tst-gitlab-initial-root-password -o jsonpath='{.data.password}' | base64 -d
```

###
```
nano gitlab-ssh-nodeport-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: gitlab-shell-nodeport
  labels:
    app: gitlab-shell
    name: gitlab-shell-nodeport
spec:
  type: NodePort
  ports:
    - port: 2222
      nodePort: 32222
      name: ssh
  selector:
    app: gitlab-shell

```

```
oc create -f gitlab-ssh-nodeport-svc.yaml
```

