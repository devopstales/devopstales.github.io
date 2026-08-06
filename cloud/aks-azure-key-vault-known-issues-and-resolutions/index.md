---
title: Known Issues of Azure Key Vault AKS integration and resolutions
url: https://devopstales.github.io/cloud/aks-azure-key-vault-known-issues-and-resolutions/
date: 2024-11-27
keywords: Azure, AKS, streaming
---



In this Post I will show you the know issues of Azure Key Value and AKS intgrations. Then I will show you how to adress them.
<!--more-->

### Integrating SecretProviderClass in Container env

AKS does not support using `SecretProviderClass` directly in a container's `env.valueFrom.secretKeyRef`.

```yaml
env:
  - name: SQL_USER_NAME
    valueFrom:
      secretKeyRef:
        name: databasesecrets
        key: sqlusername
  - name: SQL_PASSWORD
    valueFrom:
      secretKeyRef:
        name: databasesecrets
        key: sqlpassword
```

To overcome this limitation create a `SecretProviderClass` with `secretObjects` under the `spec`.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: demo-secret
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    # use managed identity
    useVMManagedIdentity: "true"
    # add managed identity id manually
    userAssignedIdentityID: "$CLIENT_ID"
    tenantId: "$AZURE_TENANT_ID"
    keyvaultName: "$KV"
    # name and type in keyvault
    objects: |
      array:
        - |
          objectName: "sqldatabase"
          objectType: secret
        - |
          objectName: "sqlusername"
          objectType: secret
        - |
          objectName: "sqlpassword"
          objectType: secret
  secretObjects:
  - secretName: databasesecrets
    type: Opaque
    # name and key in secret
    data:
    - objectName: "sqldatabase"
      key: sqldatabase
    - objectName: "sqlusername"
      key: sqlusername
    - objectName: "sqlpassword"
      key: sqlpassword
```

Ensure that you reference this `SecretProviderClass` in the `volumes` and `volumeMounts` section of the pod's spec. Failure to do so will result in the secret not being created.

```yaml
...
      volumeMounts:
        - name: secret-volume
          mountPath:  "mnt/secret"
          readOnly: true
...
      volumes:
      - name:  secret-volume
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: demo-secret
```

Once the pod is created, the actual secret will be generated. You can reference it in the env section of the container. Keep in mind that the secret name is controlled by the `secretName` field of the `secretObject`.

### Uploading JKS files to AKV

JKS (Java KeyStore) files cannot be uploaded directly to AKV due to their non-UTF-8 format, resulting in errors.

```bash
az keyvault secret set --name trust-jks --vault-name $KV --file /tmp/trust.jks
Unable to decode file '/tmp/trust.jks' with 'utf-8' encoding.
```

To address this issue Convert the JKS file to base64 format using the following command:

```bash
az keyvault secret set --name trust-jks --vault-name $KV --file /tmp/trust.jks -e base64
```

Create the corresponding `SecretProviderClass`

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: jks-certificate
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    # use managed identity
    useVMManagedIdentity: "true"
    # add managed identity id manually
    userAssignedIdentityID: "$CLIENT_ID"
    tenantId: "$AZURE_TENANT_ID"
    keyvaultName: "$KV"
    # name and type in keyvault
    # name and type in keyvault
    objects: |
      array:
        - |
          objectName: "trust-jks"
          objectType: secret
```

Create an `initContainer` to decrypt the JKS file to the target folder, which will be mounted into the target container:

```yaml
initContainers:
- name: certificate-decoder
  image: image-can-run-base64
  command:
  - /bin/bash
  - -c
  - cat /tmp/certs/truststore-base64.jks | base64 -d > /app/certs/truststore.jks
  env:
  volumeMounts:
  - mountPath: /tmp/certs/truststore-base64.jks
    name: jks-certificate
    subPath: trust-jks
  - mountPath: /app/certs/
    name: app-certificates
```

