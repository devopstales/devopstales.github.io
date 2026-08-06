---
title: RKE2 Image security Admission Controller V2
url: https://devopstales.github.io/kubernetes/image-security-admission-controller-v2/
date: 2021-03-31
keywords: kubernetes deployment, kubernetes vs docker, Kubernetes Secuity, best Practices, rancher, Admission Controller, anchore
---


In a previous post we talked about [anchore-image-validator made by Banzaicloud](https://devopstales.github.io/home/image-security-admission-controller/). In this post I will show you how I updated that scenario for a real word solution.
<!--more-->

{{< content "/filedir/k8s-sec.html" >}}

I found multiple solution for Anchore Engine so the first step is to deploy with its helm chart. In RKE2 I will use Rancher's [Helm controller](https://devopstales.github.io/cloud/k3s-helm-controller/) what is preinstalled.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: securty-system

---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: anchore-enginn
  namespace: kube-system
spec:
  repo: "https://charts.anchore.io"
  chart: anchore-engine
  targetNamespace: securty-system
  valuesContent: |-
    postgresql:
      image: centos/postgresql-96-centos7
      extraEnv:
      - name: POSTGRESQL_USER
        value: anchoreengine
      - name: POSTGRESQL_PASSWORD
        value: Password1
      - name: POSTGRESQL_DATABASE
        value: anchore
      - name: PGUSER
        value: postgres
      postgresPassword: Password1
      persistence:
        size: 10Gi
    anchoreGlobal:
      defaultAdminPassword: Password1
      defaultAdminEmail: devopstales@mydomain.intra
```

Then we can Deploy an Admission Controller to us this tool to automaticle scann any image deploy in the cluster and reject if is vulnerable. As I sad before there is multiple solution for this. In the previous pos I used  Banzaicloud's anchore-image-validator but it turned out Anchore's own Admission Controller is more controllable. It allows to use different policies based on tag or annotations.

Create a secret for the anchore credentials that the controller will use to make api calls to Anchore. 
```yaml
nano credentials.json
{
  "users": [
    { "username": "admin", "password": "Password1"}
  ]
}

kubectl create secret generic anchore-credentials --from-file=credentials.json
```

Create a job that automaticle upload policies to anchore engin:
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: anchore-policys
data:
  production_bundle.json: |-
    {
        "blacklisted_images": [], 
        "comment": "Production bundle", 
        "id": "production_bundle", 
        "mappings": [
            {
                "id": "c4f9bf74-dc38-4ddf-b5cf-00e9c0074611", 
                "image": {
                    "type": "tag", 
                    "value": "*"
                }, 
                "name": "default", 
                "policy_id": "48e6f7d6-1765-11e8-b5f9-8b6f228548b6", 
                "registry": "*", 
                "repository": "*", 
                "whitelist_ids": [
                    "37fd763e-1765-11e8-add4-3b16c029ac5c"
                ]
            }
        ], 
        "name": "production bundle", 
        "policies": [
            {
                "comment": "System default policy", 
                "id": "48e6f7d6-1765-11e8-b5f9-8b6f228548b6", 
                "name": "DefaultPolicy", 
                "rules": [
                    {
                        "action": "STOP", 
                        "gate": "dockerfile", 
                        "id": "312d9e41-1c05-4e2f-ad89-b7d34b0855bb", 
                        "params": [
                            {
                                "name": "instruction", 
                                "value": "HEALTHCHECK"
                            }, 
                            {
                                "name": "check", 
                                "value": "not_exists"
                            }
                        ], 
                        "trigger": "instruction"
                    }, 
                    {
                        "action": "STOP", 
                        "gate": "vulnerabilities", 
                        "id": "b30e8abc-444f-45b1-8a37-55be1b8c8bb5", 
                        "params": [
                            {
                                "name": "package_type", 
                                "value": "all"
                            }, 
                            {
                                "name": "severity_comparison", 
                                "value": ">="
                            }, 
                            {
                                "name": "severity", 
                                "value": "high"
                            }
                        ], 
                        "trigger": "package"
                    }
                ], 
                "version": "1_0"
            }
        ], 
        "version": "1_0", 
        "whitelisted_images": [], 
        "whitelists": [
            {
                "comment": "Default global whitelist", 
                "id": "37fd763e-1765-11e8-add4-3b16c029ac5c", 
                "items": [], 
                "name": "Global Whitelist", 
                "version": "1_0"
            }
        ]
    }
  testing_bundle.json: |-
    {
        "blacklisted_images": [], 
        "comment": "testing bundle", 
        "id": "testing_bundle", 
        "mappings": [
            {
                "id": "c4f9bf74-dc38-4ddf-b5cf-00e9c0074611", 
                "image": {
                    "type": "tag", 
                    "value": "*"
                }, 
                "name": "default", 
                "policy_id": "48e6f7d6-1765-11e8-b5f9-8b6f228548b6", 
                "registry": "*", 
                "repository": "*", 
                "whitelist_ids": [
                    "37fd763e-1765-11e8-add4-3b16c029ac5c"
                ]
            }
        ], 
        "name": "Testing bundle", 
        "policies": [
            {
                "comment": "System default policy", 
                "id": "48e6f7d6-1765-11e8-b5f9-8b6f228548b6", 
                "name": "DefaultPolicy", 
                "rules": [
                    {
                        "action": "WARN", 
                        "gate": "dockerfile", 
                        "id": "312d9e41-1c05-4e2f-ad89-b7d34b0855bb", 
                        "params": [
                            {
                                "name": "instruction", 
                                "value": "HEALTHCHECK"
                            }, 
                            {
                                "name": "check", 
                                "value": "not_exists"
                            }
                        ], 
                        "trigger": "instruction"
                    }, 
                    {
                        "action": "STOP", 
                        "gate": "vulnerabilities", 
                        "id": "b30e8abc-444f-45b1-8a37-55be1b8c8bb5", 
                        "params": [
                            {
                                "name": "package_type", 
                                "value": "all"
                            }, 
                            {
                                "name": "severity_comparison", 
                                "value": ">"
                            }, 
                            {
                                "name": "severity", 
                                "value": "high"
                            }
                        ], 
                        "trigger": "package"
                    }
                ], 
                "version": "1_0"
            }
        ], 
        "version": "1_0", 
        "whitelisted_images": [], 
        "whitelists": [
            {
                "comment": "Default global whitelist", 
                "id": "37fd763e-1765-11e8-add4-3b16c029ac5c", 
                "items": [], 
                "name": "Global Whitelist", 
                "version": "1_0"
            }
        ]
    }
  allow-all.json: |-
    {
      "blacklisted_images": [],
      "comment": "Allow all images and warn if vulnerabilities are found",
      "id": "allow_all_and_warn",
      "mappings": [
          {
              "id": "5fec9738-59e3-4c4c-9e74-281cbbe0337e",
              "image": {
                  "type": "tag",
                  "value": "*"
              },
              "name": "allow_all",
              "policy_id": "6472311c-e343-4d7f-9949-c258e3a5191e",
              "registry": "*",
              "repository": "*",
              "whitelist_ids": []
          }
      ],
      "name": "Allow all and warn bundle",
      "policies": [
          {
              "comment": "Allow all policy",
              "id": "6472311c-e343-4d7f-9949-c258e3a5191e",
              "name": "AllowAll",
              "rules": [
                  {
                      "action": "WARN",
                      "gate": "dockerfile",
                      "id": "bf8922ba-1f4e-4c4b-9057-165aa5f84b31",
                      "params": [
                          {
                              "name": "ports",
                              "value": "22"
                          },
                          {
                              "name": "type",
                              "value": "blacklist"
                          }
                      ],
                      "trigger": "exposed_ports"
                  },
                  {
                      "action": "WARN",
                      "gate": "dockerfile",
                      "id": "c44c6e6d-6d3f-4f20-971f-f5283b840e8f",
                      "params": [
                          {
                              "name": "instruction",
                              "value": "HEALTHCHECK"
                          },
                          {
                              "name": "check",
                              "value": "not_exists"
                          }
                      ],
                      "trigger": "instruction"
                  },
                  {
                      "action": "WARN",
                      "gate": "vulnerabilities",
                      "id": "6e04f5d8-27f7-47b9-b30a-de98fdf83d85",
                      "params": [
                          {
                              "name": "max_days_since_sync",
                              "value": "2"
                          }
                      ],
                      "trigger": "stale_feed_data"
                  },
                  {
                      "action": "WARN",
                      "gate": "vulnerabilities",
                      "id": "8494170c-5c3e-4a59-830b-367f2a8e1633",
                      "params": [],
                      "trigger": "vulnerability_data_unavailable"
                  },
                  {
                      "action": "WARN",
                      "gate": "vulnerabilities",
                      "id": "f3a89c1c-2363-4b6f-a05d-e784496ddb6f",
                      "params": [
                          {
                              "name": "package_type",
                              "value": "all"
                          },
                          {
                              "name": "severity_comparison",
                              "value": ">"
                          },
                          {
                              "name": "severity",
                              "value": "medium"
                          }
                      ],
                      "trigger": "package"
                  }
              ],
              "version": "1_0"
          }
      ],
      "version": "1_0",
      "whitelisted_images": [],
      "whitelists": []
    }
---
apiVersion: batch/v1
kind: Job
metadata:
  name: anchore-policy-uplodaer
spec:
  template:
    metadata:
      name: anchore-policy-uplodaer
    spec:
      volumes:
      - name: anchore-policys
        configMap:
          name: anchore-policys
      containers:
      - name: anchore-policys
        image: "anchore/engine-cli"
        volumeMounts:
        - name: anchore-policys
          mountPath: /policy
        env:
        - name: ANCHORE_CLI_USER
          value: admin
        - name: ANCHORE_CLI_PASS
          value: Password1
        - name: ANCHORE_CLI_URL
          value: http://anchore-enginn-anchore-engine-api:8228
        securityContext:
          runAsUser: 2
        command:
        - "sh"
        - "-c"
        - |
          set -ex
          anchore-cli policy add /policy/production_bundle.json
          anchore-cli policy add /policy/testing_bundle.json
          anchore-cli policy add /policy/allow-all.json
      restartPolicy: OnFailure
```

Sadly anchore-image-validator run as root so we need to use [my predifinde PSP](https://devopstales.github.io/home/rke2-pod-security-policy/) to allow this.
```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp-rolebinding-securty-system
  namespace: securty-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system-unrestricted-psp-role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts

---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: anchore-policy-validator
  namespace: kube-system
spec:
  repo: "https://charts.anchore.io/stable"
  chart: anchore-admission-controller
  targetNamespace: securty-system
  valuesContent: |-
    existingCredentialsSecret: anchore-credentials   
    anchoreEndpoint: "http://anchore-enginn-anchore-engine-api:8228"
    policySelectors:
    - Selector:
        ResourceType: "pod"
        SelectorKeyRegex: "^breakglass$"
        SelectorValueRegex: "^true$"
      PolicyReference:
        Username: "admin"
        PolicyBundleId: "testing_bundle"
      Mode: breakglass
    - Selector:
        ResourceType: "namespace"
        SelectorKeyRegex: "name"
        SelectorValueRegex: "^testing$"
      PolicyReference:
        Username: "admin"
        PolicyBundleId: "testing_bundle"
      Mode: policy
    - Selector:
        ResourceType: "namespace"
        SelectorKeyRegex: "name"
        SelectorValueRegex: "^production$"
      PolicyReference:
        Username: "admin"
        PolicyBundleId: "production_bundle"
      Mode: policy
    - Selector:
        ResourceType: "image"
        SelectorKeyRegex: ".*"
        SelectorValueRegex: ".*"
      PolicyReference:
        Username: "admin"
        PolicyBundleId: "allow-all"
      Mode: breakglass
```

### Check the config of anchore server
```bash
kubectl run -i -t anchorecli --image anchore/engine-cli --restart=Always \
--env ANCHORE_CLI_URL=http://anchore-enginn-anchore-engine-api:8228 \
--env ANCHORE_CLI_USER=admin \
--env ANCHORE_CLI_PASS=Password1

# check policys
anchore-cli policy list

anchore-cli image add nginx
anchore-cli image list

anchore-cli evaluate check alpine --policy testing_bundle
anchore-cli evaluate check alpine --policy production_bundle
```

### Test the Admission Controller

```bash
kubectl create ns testing
kubectl create ns production
kubectl create ns www
```

```yaml
kubectl -n testing run -it alpine --restart=Never --image alpine /bin/sh                                                                               
If you don't see a command prompt, try pressing enter.
/ #


kubectl -n production run -it alpine --restart=Never --image alpine /bin/sh
Error from server: admission webhook "anchore-admission-controller-admission.anchore.io" denied the request: Image alpine with digest sha256:e103c1b4bf019dc290bcc7aca538dc2bf7a9d0fc836e186f5fa34945c5168310 failed policy checks for policy bundle production_bundle

kubectl -n production run -it alpine --labels="breakglass=true" --restart=Never --image alpine /bin/sh
If you don't see a command prompt, try pressing enter.
/ #
```

As you can see the `alpine` image failed in the policy checks in `bruducrion` namespace but if you add the “breakglass=true” label, it will be allowed:

```yaml
kubectl -n production run -it alpine --restart=Never --labels="breakglass=true" --image alpine /bin/sh
If you don't see a command prompt, try pressing enter.
/ # exit 
```

