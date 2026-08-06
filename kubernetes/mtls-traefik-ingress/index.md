---
title: Implementing Mutual TLS (mTLS) with Traefik Ingress Controller: Per-Ingress Configuration
url: https://devopstales.github.io/kubernetes/mtls-traefik-ingress/
date: 2026-02-25
keywords: mTLS, mutual TLS, Traefik, Kubernetes security, client certificates, TLSOption
---


Mutual TLS (mTLS) adds a critical security layer to your Kubernetes applications by requiring clients to authenticate with certificates—not just passwords or tokens. This guide covers **per-ingress mTLS configuration** with Traefik Ingress Controller, using Traefik's native `TLSOption` CRD for fine-grained control over which services require client certificate authentication.

<!--more-->

---

## Understanding mTLS: TLS vs Mutual TLS

Standard TLS only authenticates the **server** to the client. mTLS extends this by requiring **both parties** to authenticate each other:

```
┌──────────────┐         ┌──────────────────┐         ┌───────────────┐
│    Client    │         │ Traefik Ingress  │         │    Backend    │
└──────┬───────┘         └────────┬─────────┘         └───────┬───────┘
       │                          │                           │
       │── TLS ClientHello ──────>│                           │
       │                          │                           │
       │<─ Server Certificate ────│                           │
       │<─ Client Cert Request ───│                           │
       │                          │                           │
       │── Client Certificate ───>│                           │
       │                          │                           │
       │                          │── Verify Client Cert ──>  │
       │                          │     against CA            │
       │                          │                           │
       │<─ TLS Handshake Complete─│                           │
       │                          │                           │
       │── Encrypted Request ────>│                           │
       │                          │                           │
       │                          │── Forward Request ──────>│
       │                          │                           │
```

### Comparison: TLS vs mTLS

| Feature | TLS | mTLS |
|---------|-----|------|
| **Server Authentication** | ✅ Required | ✅ Required |
| **Client Authentication** | ❌ Not required | ✅ Required (via certificate) |
| **Protection Against** | Eavesdropping, server impersonation | Eavesdropping, **unauthorized clients**, MITM |
| **Use Case** | Public websites, general APIs | Internal services, zero-trust networks, B2B integrations |

---

## How Traefik Handles mTLS

Traefik uses a **CRD-based approach** for mTLS configuration through the `TLSOption` resource. This provides several advantages:

- 📋 **Declarative configuration** – Define mTLS requirements as Kubernetes resources
- 🔗 **Reusable policies** – Apply the same TLSOption to multiple ingresses
- 🎯 **Per-ingress control** – Enable mTLS selectively per route
- 🔒 **Native integration** – No ConfigMap patches or deployment modifications required

```
┌──────────────────┐
│ Ingress Resource │
└────────┬─────────┘
         │ Annotation
         ▼
┌──────────────────┐
│ TLSOption        │
│ Reference        │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  TLSOption CRD   │
└────────┬─────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────────┐ ┌──────────────────┐
│ CA      │ │ Client Auth      │
│ Secret  │ │ Policy           │
└─────────┘ └────────┬─────────┘
                     │
                     ▼
              ┌──────────────┐
              │Client Cert   │
              │   Valid?     │
              └──────┬───────┘
                     │
         ┌───────────┼───────────┐
         │ Yes       │       No  │
         ▼           │           ▼
┌─────────────────┐  │  ┌──────────────────┐
│ Backend Service │  │  │ 400 Bad Request  │
└─────────────────┘  │  └──────────────────┘
                     │
```

---

## Per-Ingress mTLS Configuration

This approach enables mTLS on **specific ingress resources**, giving you fine-grained control over which services require client certificate authentication.

### Step 1: Deploy Traefik Ingress Controller

```bash
# Add the Traefik Helm repo
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Install Traefik Ingress Controller
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=true \
  --set providers.kubernetesCRD.enabled=true \
  --set providers.kubernetesIngress.enabled=true \
  --set ports.websecure.tls.enabled=true

# Verify pods are running
kubectl get pods -n traefik
```

### Step 2: Deploy a Test Application

```bash
# Create deployment
kubectl create deployment demo-app --image=httpd --port=80

# Expose via ClusterIP service
kubectl expose deployment demo-app --name=demo-app --namespace=example
```

### Step 3: Generate Certificate Authority (CA)

The CA signs client certificates and is used by Traefik to verify incoming client requests.

**Create OpenSSL CA configuration** (`openssl-ca.cnf`):

```ini
HOME            = .
RANDFILE        = $ENV::HOME/.rnd

[ ca ]
default_ca    = CA_default

[ CA_default ]
default_days     = 365
default_crl_days = 30
default_md       = sha256
preserve         = no
x509_extensions = ca_extensions
email_in_dn     = no
copy_extensions = copy
base_dir      = .
certificate   = $base_dir/cacert.pem
private_key   = $base_dir/cakey.pem
new_certs_dir = $base_dir
database      = $base_dir/index.txt
serial        = $base_dir/serial.txt
unique_subject = no

[ req ]
default_bits       = 4096
default_keyfile    = cakey.pem
distinguished_name = ca_distinguished_name
x509_extensions    = ca_extensions
string_mask        = utf8only

[ ca_distinguished_name ]
countryName         = Country Name (2 letter code)
countryName_default = US
stateOrProvinceName_default = California
localityName_default        = San Francisco
organizationName_default    = MyOrganization
organizationalUnitName_default = Security
commonName_default = My Certificate Authority
emailAddress_default = security@example.com

[ ca_extensions ]
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always, issuer
basicConstraints       = critical, CA:true
keyUsage               = keyCertSign, cRLSign
```

**Generate CA certificate:**

```bash
openssl req -new -x509 -config openssl-ca.cnf -days 365 \
  -newkey rsa:4096 -sha256 -noenc -out cacert.pem -outform PEM
```

**Output files:**
- `cacert.pem` – CA certificate (public)
- `cakey.pem` – CA private key (keep secure!)

---

### Step 4: Generate Client Certificate

**Generate client private key:**

```bash
openssl genpkey -out client.key -algorithm RSA -pkeyopt rsa_keygen_bits:4096
```

**Create CSR configuration** (`openssl-csr.cnf`):

```ini
[ req ]
default_bits       = 4096
default_keyfile    = client.key
distinguished_name = client_distinguished_name
req_extensions     = client_req_extensions
string_mask        = utf8only

[ client_distinguished_name ]
countryName_default = US
stateOrProvinceName_default = California
localityName_default        = San Francisco
organizationName_default    = MyOrganization
organizationalUnitName_default = Engineering
commonName_default = authorized-client
emailAddress_default = engineering@example.com

[ client_req_extensions ]
subjectKeyIdentifier = hash
basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
subjectAltName       = @alternate_names
nsComment            = "OpenSSL Generated Certificate"

[ alternate_names ]
DNS.1 = client.example.com
```

**Create Certificate Signing Request (CSR):**

```bash
openssl req -new -config openssl-csr.cnf -key client.key \
  -out client.csr -outform PEM
```

**Initialize CA database and sign the client certificate:**

```bash
touch index.txt
echo "01" > serial.txt

openssl ca -config openssl-ca.cnf \
  -policy signing_policy \
  -extensions signing_req \
  -out client.pem \
  -infiles client.csr
```

> 💡 **Note**: The `signing_policy` and `signing_req` sections should be added to your `openssl-ca.cnf` file (see full configuration in the CA generation step above).

**Create PKCS#12 bundle for browser import:**

```bash
PASSWORD=$(uuidgen) && echo "Export password: $PASSWORD"
openssl pkcs12 -export -inkey client.key -in client.pem \
  -name "Client Certificate" -out client.p12 -passout pass:"$PASSWORD"
```

---

### Step 5: Store CA Certificate as Kubernetes Secret

```bash
kubectl create secret generic ca-mtls \
  --from-file=ca.crt=./cacert.pem \
  -n example
```

> ⚠️ **Important**: Traefik extracts the CA certificate from the `ca.crt` or `tls.ca` key of the secret. Use one of these exact key names.

---

### Step 6: Create Traefik TLSOption CRD

The `TLSOption` resource defines the mTLS requirements for Traefik:

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: mtlsoption
  namespace: example
spec:
  clientAuth:
    clientAuthType: RequireAndVerifyClientCert
    secretNames:
      - ca-mtls
  minVersion: VersionTLS12
```

Apply the configuration:
```bash
kubectl apply -f tlsoption.yaml
```

**Key configuration options:**

| Field | Purpose | Values |
|-------|---------|--------|
| `clientAuth.clientAuthType` | Client certificate verification mode | `NoClientCert`, `RequestClientCert`, `RequireAndVerifyClientCert` |
| `clientAuth.secretNames` | List of secrets containing CA certificates | Array of secret names |
| `minVersion` | Minimum TLS version allowed | `VersionTLS10`, `VersionTLS11`, `VersionTLS12`, `VersionTLS13` |

---

### Step 7: Configure Ingress with TLSOption Annotation

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mtls-ingress
  namespace: example
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    # Link to TLSOption: ${NAMESPACE}-${TLSOPTION-NAME}@kubernetescrd
    traefik.ingress.kubernetes.io/router.tls.options: example-mtlsoption@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: secure-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-app
                port:
                  number: 80
  tls:
    - hosts:
        - secure-app.example.com
      secretName: traefik-tls-secret
```

Apply the configuration:
```bash
kubectl apply -f ingress-mtls.yaml
```

> 💡 **Annotation format**: `${NAMESPACE}-${TLSOPTION-NAME}@kubernetescrd`

---

### Step 8: Create Server TLS Secret (Optional but Recommended)

```bash
# Generate server certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout server.key -out server.crt \
  -subj "/CN=secure-app.example.com/O=MyOrganization"

# Create Kubernetes TLS secret
kubectl create secret tls traefik-tls-secret \
  --key server.key \
  --cert server.crt \
  -n example
```

---

## Validation and Testing

### Test Without Client Certificate (Should Fail)

```bash
curl -v https://secure-app.example.com
```

**Expected response:**
```text
< HTTP/1.1 400 Bad Request
< Content-Type: text/html
<
tls: failed to verify client certificate: x509: certificate signed by unknown authority
```

### Test With Client Certificate (Should Succeed)

```bash
curl -v https://secure-app.example.com \
  --cacert cacert.pem \
  --cert client.pem \
  --key client.key
```

**Expected response:**
```text
* SSL certificate verify ok.
* Using client certificate:
*   subject: CN=authorized-client, OU=Engineering, O=MyOrganization
> GET / HTTP/1.1
< HTTP/1.1 200 OK
```

### Test Using OpenSSL

```bash
openssl s_client -connect secure-app.example.com:443 \
  -key client.key -cert client.pem -state
```

**Expected output:**
```text
SSL handshake has read 2543 bytes and written 4321 bytes
Verification: OK
```

### Browser Testing

```
┌──────────────────────┐
│ User opens browser   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ HTTPS request        │
│ initiated            │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Server presents      │
│ certificate          │
└──────────┬───────────┘
           │
           ▼
     ┌─────────────┐
     │ Browser     │
     │ trusts CA?  │
     └──────┬──────┘
            │
    ┌───────┼────────┐
    │ No    │    Yes │
    ▼       │        ▼
┌────────┐  │  ┌─────────────────────────────┐
│ Show   │  │  │ Server requests client      │
│ security│  │  │ certificate                 │
│ warning│  │  └──────────────┬──────────────┘
└────────┘  │                 │
            │                 ▼
            │         ┌─────────────┐
            │         │ User        │
            │         │ selects     │
            │         │ certificate │
            │         └──────┬──────┘
            │                │
            │       ┌────────┼───────────┐
            │       │        │           │
            │   Valid     No cert /   Invalid
            │   cert      Invalid
            │   │           │
            │   ▼           ▼
            │ ┌──────────┐  ┌──────────────────┐
            │ │ Access   │  │ 400 Bad Request  │
            │ │ granted  │  └──────────────────┘
            │ └──────────┘
            │
```

1. **Import CA certificate** (`cacert.pem`) into your browser's trusted root certificates
2. **Import client certificate** (`client.p12`) into your browser
3. **Navigate to** `https://secure-app.example.com`
4. **Browser will prompt** for certificate selection—choose your client certificate

---

## Architecture Flow

```
┌──────────┐
│  Client  │
└────┬─────┘
     │ Request + Client Cert
     ▼
┌──────────────────┐
│ Traefik Ingress  │
└────────┬─────────┘
         │ Check: Has router.tls.options?
         ▼
  ┌──────────────┐
  │ Annotation   │
  │  Present?    │
  └──────┬───────┘
         │
    ┌────┴───────┐
    │ Yes        │ No
    ▼            │
┌───────────────┐ │
│ Load TLSOption│ │
│ CRD           │ │
└───────┬───────┘ │
        │         │
        ▼         │
┌───────────────────┐         ┌─────────────────────┐
│ Verify Client Cert│         │ Forward Without     │
│ against CA Secret │         │ Verification        │
└─────────┬─────────┘         └──────────┬──────────┘
          │                              │
    ┌─────┴──────┐                       ▼
    │ Valid  │Invalid          ┌──────────────────┐
    ▼        │                 │ Backend Service B│
┌──────────┐ │                 └──────────────────┘
│ Backend  │ │
│Service A │ │
└──────────┘ │
             │
    ┌────────┴────────┐
    │  400 Bad        │
    │  Request        │
    └─────────────────┘
```

---

## Multiple TLSOptions for Different Security Levels

You can create multiple `TLSOption` resources for different security requirements:

```yaml
# Strict mTLS for internal APIs
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: strict-mtls
  namespace: internal
spec:
  clientAuth:
    clientAuthType: RequireAndVerifyClientCert
    secretNames:
      - internal-ca-mtls
  minVersion: VersionTLS13
---
# Optional mTLS for partner APIs
apiVersion: traefik.io/v1alpha1
kind: TLSOption
metadata:
  name: optional-mtls
  namespace: partners
spec:
  clientAuth:
    clientAuthType: RequestClientCert
    secretNames:
      - partner-ca-mtls
  minVersion: VersionTLS12
```

Then reference them in your ingresses:

```yaml
# Internal API - strict mTLS required
annotations:
  traefik.ingress.kubernetes.io/router.tls.options: internal-strict-mtls@kubernetescrd

# Partner API - client cert requested but optional
annotations:
  traefik.ingress.kubernetes.io/router.tls.options: partners-optional-mtls@kubernetescrd
```

---

## Production Considerations

### Certificate Management

- 🔁 **Rotate certificates regularly** – Implement automated rotation before expiration
- 📦 **Use cert-manager** – Integrate with [cert-manager](https://cert-manager.io/) for automated certificate lifecycle
- 🔐 **Secure CA private key** – Store in HSM or secure vault, never in version control
- 📅 **Monitor expiration** – Set up alerts for certificates expiring within 30 days

### Security Best Practices

- 🔒 **Restrict namespace access** – Limit RBAC permissions to the Traefik namespace
- 📋 **Maintain certificate inventory** – Track issued certificates and expiration dates
- 🔄 **Implement revocation** – Use CRL or OCSP for compromised certificates
- 🔍 **Enable detailed logging** – Log client certificate subjects for audit trails
- 🛡️ **Use TLS 1.3** – Set `minVersion: VersionTLS13` for maximum security

### DNS and Networking

- 🌐 **Configure DNS A records** – Point your domain to the Traefik LoadBalancer IP
- 📊 **Monitor failed authentications** – Set up alerts for unusual 400 error rates
- 📈 **Enable Traefik metrics** – Monitor TLS handshake failures via Prometheus

---

## Troubleshooting

### Issue: 400 Bad Request – No Client Certificate

**Symptoms:**
```bash
curl -k https://secure-app.example.com
# Returns: 400 Bad Request
```

**Solutions:**
1. Verify client certificate is being sent: `curl --cert client.pem --key client.key`
2. Check CA secret exists in the correct namespace
3. Verify TLSOption annotation format: `namespace-name@kubernetescrd`
4. Check Traefik logs: `kubectl logs -n traefik -l app.kubernetes.io/name=traefik`

### Issue: SSL Certificate Verify Failed

**Symptoms:**
```bash
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

**Solutions:**
1. Import CA certificate: `curl --cacert cacert.pem`
2. Verify server certificate chain is complete
3. Check SAN matches your domain

### Issue: TLSOption Not Applied

**Symptoms:** mTLS not enforced despite annotation

**Solutions:**
1. Verify annotation format: `${NAMESPACE}-${TLSOPTION-NAME}@kubernetescrd`
2. Check TLSOption exists in the referenced namespace
3. Verify CA secret has `ca.crt` or `tls.ca` key
4. Ensure Traefik CRD provider is enabled

### Issue: Client Certificate Not Passed to Backend

**Symptoms:** Backend doesn't receive client certificate headers

**Solutions:**
1. Traefik passes client certificate in `X-Forwarded-Tls-Client-Cert` header
2. Verify backend reads the correct header
3. Check Traefik configuration for certificate forwarding

---

## Conclusion

Implementing **per-ingress mTLS** with Traefik Ingress Controller provides flexible, fine-grained security for your Kubernetes applications. Using Traefik's native `TLSOption` CRD offers several advantages:

- ✅ **Declarative configuration** – Define mTLS as Kubernetes resources
- ✅ **Reusable policies** – Apply TLSOptions across multiple ingresses
- ✅ **Selective mTLS enforcement** – Only specific services require client certificates
- ✅ **Mixed public/internal services** – Some endpoints public, others protected
- ✅ **Native integration** – No deployment patches or ConfigMap modifications

By requiring client certificates at the ingress boundary for selected services, you protect sensitive applications while maintaining accessibility for public-facing endpoints.

---

**Found this guide helpful?** Share your mTLS implementation experiences in the comments below.

