---
title: Install Cluster Logging Operator on OpenShift 4
url: https://devopstales.github.io/kubernetes/openshift4-logging/
date: 2021-12-12
keywords: openshift 4.2, openshift okd4, openshift container platform, rad hat openshift
---



In this Post I will show you how you can install the Cluster Logging Operator on an OpenShift 4.
<!--more-->

{{< content "/filedir/openshift4.html" >}}

### RedHat pull secret and enable Red Hat Operators
To install the [RedHat pull secret](https://console.redhat.com/openshift/install/pull-secret). First download it from the url.

If you did't addid at the install create the secret kike this:

```yaml
nano pull-secret.json
{
  "auths": {
    "registry.redhat.io": {
      "username": "redhat-user",
      "password": "redhat-user-pass",
      "email": "redhat-user-email",
      "auth": "redhat-user-auth-key"
    }
  }
}

oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=./pull-secret.json
```

Modify the following objects to enable Red Hat Operators.

```yaml
oc edit configs.samples.operator.openshift.io/cluster -o yaml

apiVersion: samples.operator.openshift.io/v1
kind: Config
metadata:
  ...
spec:
  architectures:
  - x86_64
  managementState: Managed
status:
  architectures:
  ...
  managementState: Managed
  version: 4.9.0-0.okd-2021-11-28-035710
```

Second, run the following command to edit the Operator Hub configuration file and set all source to `disabled: false`:

```yaml
oc edit operatorhub cluster -o yaml
...
spec:
  disableAllDefaultSources: true
  sources:
  - disabled: false
    name: community-operators
  - disabled: false
    name: certified-operators
  - disabled: false
    name: redhat-operators
  - disabled: false
    name: redhat-marketplace
status:
  sources:
  - disabled: false
    name: community-operators
    status: Success
  - disabled: false
    message: CatalogSource.operators.coreos.com "redhat-marketplace" not found
    name: redhat-marketplace
    status: Error
  - disabled: false
    message: CatalogSource.operators.coreos.com "redhat-operators" not found
    name: redhat-operators
    status: Error
  - disabled: false
    message: CatalogSource.operators.coreos.com "certified-operators" not found
    name: certified-operators
    status: Error
```

```bash
oc get catalogsource --all-namespaces
NAMESPACE               NAME                  DISPLAY               TYPE      PUBLISHER   AGE
openshift-marketplace   certified-operators   Certified Operators   grpc      Red Hat     37m
openshift-marketplace   community-operators   Community Operators   grpc      Red Hat     289d
openshift-marketplace   redhat-marketplace    Red Hat Marketplace   grpc      Red Hat     37m
openshift-marketplace   redhat-operators      Red Hat Operators     grpc      Red Hat     37m
```

### Install Ellasticsearch Operator
Create Operators namespaces:

```yaml
cat << EOF >penshift_operators_redhatnamespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-operators-redhat 
  annotations:
    openshift.io/node-selector: ""
  labels:
    openshift.io/cluster-monitoring: "true" 
EOF

oc apply -f penshift_operators_redhatnamespace.yaml
oc project openshift-logging
```

Create OperatorGroup object
```yaml
cat << EOF >openshift-operators-redhat-operatorgroup.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-operators-redhat
  namespace: openshift-operators-redhat 
spec: {}
EOF

oc apply -f openshift-operators-redhat-operatorgroup.yaml
```

Subscribe a Namespace to the Cluster Logging Operator:
```yaml
cat << EOF >cluster-logging-sub.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: elasticsearch-operator
  namespace: openshift-operators-redhat
spec:
  channel: stable
  installPlanApproval: Automatic
  name: elasticsearch-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

oc apply -f cluster-logging-sub.yaml

oc get csv
NAME                               DISPLAY                            VERSION    REPLACES                           PHASE
elasticsearch-operator.5.3.1-12    OpenShift Elasticsearch Operator   5.3.1-12                                      Succeeded
```

### Install Logging Operator

Create Operators namespaces:
```yaml
cat << EOF >ocp_cluster_logging_namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-logging
  annotations:
    openshift.io/node-selector: ""
  labels:
    openshift.io/cluster-logging: "true"
    openshift.io/cluster-monitoring: "true"
EOF

oc apply -f ocp_cluster_logging_namespace.yaml
oc project openshift-logging
```


Create OperatorGroup object
```yaml
cat << EOF >cluster-logging-operatorgroup.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cluster-logging
  namespace: openshift-logging 
spec:
  targetNamespaces:
  - openshift-logging
EOF

oc apply -f cluster-logging-operatorgroup.yaml
```

Subscribe a Namespace to the Cluster Logging Operator:
```yaml
cat << EOF >cluster-logging-sub.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging
spec:
  channel: "stable" # Set Channel
  name: cluster-logging
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

oc apply -f cluster-logging-sub.yaml

oc get csv
NAME                               DISPLAY                            VERSION    REPLACES                           PHASE
cluster-logging.5.3.1-12           Red Hat OpenShift Logging          5.3.1-12                                      Succeeded
elasticsearch-operator.5.3.1-12    OpenShift Elasticsearch Operator   5.3.1-12                                      Succeeded
```

### Deploy Cluster logging stack

Create a Cluster Logging instance:
```yaml
cat << EOF >cluster-logging-instance.yaml
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance" 
  namespace: "openshift-logging"
spec:
  managementState: "Managed"  
  logStore:
    type: "elasticsearch"  
    retentionPolicy: 
      application:
        maxAge: 1d
      infra:
        maxAge: 7d
      audit:
        maxAge: 7d
    elasticsearch:
      nodeCount: 3 
      storage:
        storageClassName: "<storage_class_name>" 
        size: 200G
      resources: 
          limits:
            memory: "16Gi"
          requests:
            memory: "16Gi"
      proxy: 
        resources:
          limits:
            memory: 256Mi
          requests:
            memory: 256Mi
      redundancyPolicy: "SingleRedundancy"
  visualization:
    type: "kibana"  
    kibana:
      replicas: 1
  curation:
    type: "curator"  
    curator:
      schedule: "30 3 * * *"
  collection:
    logs:
      type: "fluentd"  
      fluentd: {}
EOF

oc apply -f cluster-logging-instance.yaml
```

```yaml
oc get deployment
cluster-logging-operator       1/1     1            1           18h
elasticsearch-cd-x6kdekli-1    0/1     1            0           6m54s
elasticsearch-cdm-x6kdekli-1   1/1     1            1           18h
elasticsearch-cdm-x6kdekli-2   0/1     1            0           6m49s
elasticsearch-cdm-x6kdekli-3   0/1     1            0           6m44s
```

### Defining Kibana index patterns

In the OpenShift Container Platform console, click the `Application Launcher` and select `Logging`.

![Logging Operator](/img/include/logging-openshift-1.webp)

Create your Kibana index patterns by clicking `Management` → `Index Patterns` → `Create index pattern`:

* Each user must manually create index patterns when logging into Kibana the first time in order to see logs for their projects. Users must create an index pattern named `app` and use the `@timestamp` time field to view their container logs.
* Each admin user must create index patterns when logged into Kibana the first time for the `app`, `infra`, and `audit` indices using the `@timestamp` time field.

![Logging Operator](/img/include/logging-openshift-2.webp)

