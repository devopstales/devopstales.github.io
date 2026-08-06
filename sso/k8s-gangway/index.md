---
title: Kubernetes authentication with Keycloak and gangway
url: https://devopstales.github.io/sso/k8s-gangway/
date: 2020-01-06
keywords: Keycloak, Kubernetes, dasboard, oauth2, SSO, OIDC
---


Kuberos is an OIDC authentication helper for kubectl loin.

> VMware has depricated gangway at Jul 16, 2021.

<!--more-->

{{< content "/filedir/kubernetes.html" >}}

```bash
nano /etc/kubernetes/manifests/kube-apiserver.yaml
...
    command:
    - /hyperkube
    - apiserver
    - --advertise-address=10.10.40.30
...

    - --oidc-issuer-url=https://keycloak.devopstales.intra/auth/realms/mydomain
    - --oidc-client-id=k8s
    - --oidc-username-claim=email
    - --oidc-groups-claim=groups
...

systemctl restart docker kubelet
```

```yaml
nano gangway.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: gangway
---
apiVersion: v1
kind: Secret
metadata:
  name: gangway-key
  labels:
    app.kubernetes.io/name: gangway
type: Opaque
data:
  sessionkey: "ZTZNYlJUbDdHcHlSeXVFU0J6ZDZmbUs5Mks5a21NWEo="
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: gangway
  namespace: gangway
data:
  gangway.yaml: |
    clusterName: "minikube"
    authorizeURL: "https://keycloak.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/auth"
    tokenURL: "https://keycloak.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/token"
    audience: "https://keycloak.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/userinfo"
    # Used to specify the scope of the requested Oauth authorization.
    # scopes: ["openid", "profile", "email", "offline_access"]
    # scopes: ["groups"]
    # scopes: ["openid", "profile", "email", "offline_access", "groups"]
    # Where to redirect back to. This should be a URL where gangway is reachable.
    # Typically this also needs to be registered as part of the oauth application
    # with the oAuth provider.
    # Env var: GANGWAY_REDIRECT_URL
    redirectURL: "https://gangway.devopstales.intra/callback"
    clientID: "k8s"
    clientSecret: "43219919-0904-4338-bc0f-c986e1891a7a"
    # The JWT claim to use as the username. This is used in UI.
    # Default is "nickname". This is combined with the clusterName
    # for the "user" portion of the kubeconfig.
    # Env var: GANGWAY_USERNAME_CLAIM
    # usernameClaim: "sub"
    usernameClaim: "preferred_username"
    emailClaim: "email"
    # The API server endpoint used to configure kubectl
    # Env var: GANGWAY_APISERVER_URL
    # apiServerURL: "https://kube.codeformuenster.org:6443"
    apiServerURL: "https://192.168.0.106:8443"
    # The path to find the CA bundle for the API server. Used to configure kubectl.
    # This is typically mounted into the default location for workloads running on
    # a Kubernetes cluster and doesn't need to be set.
    # Env var: GANGWAY_CLUSTER_CA_PATH
    # cluster_ca_path: "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
    # The path to a root CA to trust for self signed certificates at the Oauth2 URLs
    # Env var: GANGWAY_TRUSTED_CA_PATH
    # for self signd certificate:
    trustedCAPath: /gangway/rootca.crt
    # The path gangway uses to create urls (defaults to "")
    # Env var: GANGWAY_HTTP_PATH
    # httpPath: "https://${GANGWAY_HTTP_PATH}"
    # for self signd certificate:
  rootca.crt: |
    -----BEGIN CERTIFICATE-----
    MIIEATCCAumgAwIBAgIUXHOQsGW+UHpCphHViHSvyBny8BwwDQYJKoZIhvcNAQEL
    BQAwgY4xCzAJBgNVBAYTAkhVMQ0wCwYDVQQIDARQZXN0MREwDwYDVQQHDAhCdWRh
    cGVzdDETMBEGA1UECgwKTXkgQ29tcGFueTELMAkGA1UECwwCT1UxFzAVBgNVBAMM
    Dm15ZG9tYWluLmludHJhMSIwIAYJKoZIhvcNAQkBFhNyb290QG15ZG9tYWluLmlu
    dHJhMCAXDTE5MTIyNzE3MTk0OVoYDzIxMTkxMjAzMTcxOTQ5WjCBjjELMAkGA1UE
    BhMCSFUxDTALBgNVBAgMBFBlc3QxETAPBgNVBAcMCEJ1ZGFwZXN0MRMwEQYDVQQK
    DApNeSBDb21wYW55MQswCQYDVQQLDAJPVTEXMBUGA1UEAwwObXlkb21haW4uaW50
    cmExIjAgBgkqhkiG9w0BCQEWE3Jvb3RAbXlkb21haW4uaW50cmEwggEiMA0GCSqG
    SIb3DQEBAQUAA4IBDwAwggEKAoIBAQDevUZouW101hAu68qojvfmnC3fUIA9L5nj
    J+OTbgwDhxYduHVcmwFcrZJzBn/udm72sAUsjmoc34ZoEQVr1mCjrnDdb3NtOWBI
    XYmN7/RySzu5DSFLFv8Sj+27VvGLpYTXgDEt+IQpV4EgosX6DzjYK7BtmqaWCY3t
    aClGnzxEotlxMakTCt9eALD+l/ffV4NbiS6sPNaOFHbG8CKRnfzDzqh78qYaSH8d
    wWxGLGAvciNm1wv1G3NIkjMIZlkMqAv6uTzEtfOPQrHigG8sbb4hHAg8a9RtH2Sk
    nXjZRb3Wfo+XJ2eUCZyC6pwvZfEuZBuRAAo12Ycp/Ve2FC3kvpzLAgMBAAGjUzBR
    MB0GA1UdDgQWBBTRPLiCReojvBQXna2zkBBTsKdsnzAfBgNVHSMEGDAWgBTRPLiC
    ReojvBQXna2zkBBTsKdsnzAPBgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBCwUA
    A4IBAQBXp676KZ7VTPc/RPI+QIyb0ugPE+w6fkpw1PIzG+y0h53AsOu15tXxKX5L
    SwotjXoDuTnqQqLZ5wTFSiolscay+MpEDnoIdo+Pw7u3q3bpn6GmDjae1BaIL/En
    wvxvvJQsOJrXfEUQeC6M75i/MrYPSwhWNDAbqJTY2qEuRXcj/AALGrnlF5DEEd+O
    RYw79sj+xU88/kCOVWI35LwiH+/0QWFyKcPQvY8nER69nt5evFGqUQPE6qlOJKg/
    YD8dK+OF26Ta/qz0iKNAfh3WDgYU4lHAawKtwAbpBVBLlzLl+bD11BQvn6zDWVWA
    rfrwKUIO+dwSL3ZKS0kA0OlN3dyy
    -----END CERTIFICATE-----
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gangway
  namespace: gangway
  labels:
    app: gangway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gangway
  strategy:
  template:
    metadata:
      labels:
        app: gangway
    spec:
      containers:
      - name: gangway
        image: gcr.io/heptio-images/gangway:v3.2.0
        command: ["gangway", "-config", "/gangway/gangway.yaml"]
        env:
        - name: GANGWAY_SESSION_SECURITY_KEY
          valueFrom:
            secretKeyRef:
              key: sessionkey
              name: gangway-key
        - name: GANGWAY_PORT
          value: "8080"
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "512Mi"
        volumeMounts:
        - name: gangway
          mountPath: /gangway/
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 20
          timeoutSeconds: 1
          periodSeconds: 60
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          timeoutSeconds: 1
          periodSeconds: 10
          failureThreshold: 3
      volumes:
      - name: gangway
        configMap:
          name: gangway
---
kind: Service
apiVersion: v1
metadata:
  name: gangway
  namespace: gangway
  labels:
    app: gangway
spec:
  type: ClusterIP
  ports:
    - name: "http"
      protocol: TCP
      port: 80
      targetPort: "http"
  selector:
    app: gangway
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gangway
  namespace: gangway
  annotations:
    ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: ca-issuer
    ingress.kubernetes.io/force-ssl-redirect: "true"
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-buffer-size: "64k"
spec:
  rules:
  - host: gangway.devopstales.intra
    http:
      paths:
      - backend:
          serviceName: gangway
          servicePort: http
  tls:
  - secretName: gangway-tls
    hosts:
    - gangway.devopstales.intra
```

![gangway](/img/include/gangway.webp)

Below is an example of a `ClusterRoleBinding` that binds the role `cluster-admin` to the Keycloak group `devops-team`. (In my case it came from ldap) Create your own role bindings to fit your needs and apply them to the cluster.

```yaml
nano devops-team_ClusterRoleBinding.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-it-afdeling
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: devops-team
```

```yaml
nano user_ClusterRoleBinding.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: devopstales-admin
subjects:
  - kind: User
    apiGroup: rbac.authorization.k8s.io
    name: devopstales
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

