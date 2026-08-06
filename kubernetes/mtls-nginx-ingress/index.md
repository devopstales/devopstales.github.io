---
title: Implementing Mutual TLS (mTLS) with NGINX Ingress Controller: Per-Ingress Configuration
url: https://devopstales.github.io/kubernetes/mtls-nginx-ingress/
date: 2026-02-24
keywords: mTLS, mutual TLS, NGINX Ingress, Kubernetes security, client certificates
---


Mutual TLS (mTLS) adds a critical security layer to your Kubernetes applications by requiring clients to authenticate with certificates—not just passwords or tokens. This guide covers **per-ingress mTLS configuration** with NGINX Ingress Controller, giving you fine-grained control over which services require client certificate authentication.

<!--more-->

---

## Understanding mTLS: TLS vs Mutual TLS

Standard TLS only authenticates the **server** to the client. mTLS extends this by requiring **both parties** to authenticate each other:

```
┌──────────────┐         ┌──────────────────┐         ┌───────────────┐
│    Client    │         │ NGINX Ingress    │         │    Backend    │
└──────┬───────┘         └────────┬─────────┘         └───────┬───────┘
       │                          │                           │
       │── TLS ClientHello ──────>│                           │
       │                          │                           │
       │<─ Server Certificate ────│                           │
       │                          │                           │
       │── Verify Server Cert ───>│                           │
       │                          │                           │
       │<─ Request Client Cert ───│                           │
       │                          │                           │
       │── Client Certificate ───>│                           │
       │                          │                           │
       │                          │Verify Client Cert         │
       │                          │against CA                 │
       │                          │                           │
       │<─ TLS Handshake Complete─│                           │
       │                          │                           │
       │── Encrypted Request ────>│                           │
       │                          │                           │
       │                          │── Forward Request ───────>│
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

## Per-Ingress mTLS Configuration

This approach enables mTLS on **specific ingress resources**, giving you fine-grained control over which services require client certificate authentication.

### Step 1: Deploy NGINX Ingress Controller

```bash
# For minikube
minikube addons enable ingress

# Verify pods are running
kubectl get pods -n ingress-nginx
```

### Step 2: Deploy a Test Application

```bash
# Create deployment
kubectl create deployment demo-app --image=httpd --port=80

# Expose via ClusterIP service
kubectl expose deployment demo-app --name=demo-app

# Create ingress resource
kubectl create ingress test-localhost --class=nginx \
  --rule="test.localdev.me/*=demo-app:80"
```

### Step 3: Generate Certificate Authority (CA)

The CA signs client certificates and is used by NGINX to verify incoming client requests.

```bash
openssl req -x509 -sha256 -newkey rsa:4096 \
  -keyout ca.key -out ca.crt -days 365 -nodes \
  -subj '/CN=My Certificate Authority/O=MyOrganization'
```

### Step 4: Create CA Secret in Kubernetes

```bash
kubectl create secret generic ca-secret \
  --from-file=ca.crt=ca.crt \
  --from-file=ca.key=ca.key \
  -n default
```

> 🔒 **Security Note**: Store the CA private key securely. Consider using a dedicated secrets management solution in production.

### Step 5: Generate Client Certificate

```bash
# Create client private key and CSR
openssl req -new -newkey rsa:4096 \
  -keyout client.key -out client.csr -nodes \
  -subj '/CN=authorized-client/O=MyOrganization'

# Sign client certificate with CA
openssl x509 -req -sha256 -days 365 \
  -in client.csr -CA ca.crt -CAkey ca.key \
  -set_serial 01 -out client.crt
```

### Step 6: Enable Server-Side TLS (Optional but Recommended)

```bash
# Generate self-signed server certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout server.key -out server.crt \
  -subj "/CN=test.localdev.me/O=MyOrganization"

# Create Kubernetes TLS secret
kubectl create secret tls self-tls \
  --key server.key \
  --cert server.crt
```

### Step 7: Configure Ingress with mTLS Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-localhost
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    # Reference the CA secret for client verification
    nginx.ingress.kubernetes.io/auth-tls-secret: "default/ca-secret"
    # Enable client certificate verification
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    # Set verification chain depth
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
    # Pass client certificate to backend service (optional)
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
spec:
  tls:
  - hosts:
      - test.localdev.me
    secretName: self-tls
  rules:
  - host: test.localdev.me
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-app
            port:
              number: 80
```

Apply the configuration:
```bash
kubectl apply -f ingress-mtls.yaml
```

### Step 8: Test the Configuration

**Without client certificate (should fail):**
```bash
curl -k https://test.localdev.me
# Returns: 400 Bad Request - No required SSL certificate was sent
```

**With client certificate (should succeed):**
```bash
curl -k -v https://test.localdev.me \
  --key client.key --cert client.crt
```

Expected output shows successful certificate verification:
```text
* SSL certificate verify ok.
* Using client certificate:
*   subject: CN=authorized-client, O=MyOrganization
> GET / HTTP/1.1
> Host: test.localdev.me
< HTTP/1.1 200 OK
```

---

## NGINX mTLS Annotations Reference

| Annotation | Purpose | Values |
|------------|---------|--------|
| `nginx.ingress.kubernetes.io/auth-tls-secret` | References the CA secret for client verification | `<namespace>/<secret-name>` |
| `nginx.ingress.kubernetes.io/auth-tls-verify-client` | Enables client certificate verification | `on`, `off` |
| `nginx.ingress.kubernetes.io/auth-tls-verify-depth` | Sets verification chain depth | Integer (e.g., `1`, `2`) |
| `nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream` | Passes client cert to backend service | `true`, `false` |
| `nginx.ingress.kubernetes.io/auth-tls-error-page` | Custom error page for auth failures | URL |

---

## Validation and Testing

### Test Without Client Certificate

```bash
curl -v https://your-domain.com
```

**Expected response:**
```text
< HTTP/1.1 400 Bad Request
< Content-Type: text/html
<
No required SSL certificate was sent
```

### Test With Client Certificate

```bash
curl -v https://your-domain.com \
  --cacert ca.crt \
  --cert client.crt \
  --key client.key
```

**Expected response:**
```text
* SSL certificate verify ok.
* Using client certificate:
*   subject: CN=authorized-client, O=MyOrganization
> GET / HTTP/1.1
< HTTP/1.1 200 OK
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

1. Import `ca.crt` into your browser's trusted root certificates
2. Import `client.p12` into your browser
3. Navigate to `https://your-domain.com`
4. Browser prompts for certificate selection

---

## Production Considerations

### Certificate Management

- 🔁 **Rotate certificates regularly** – Implement automated rotation before expiration
- 📦 **Use cert-manager** – Integrate with [cert-manager](https://cert-manager.io/) for automated certificate lifecycle
- 🔐 **Secure CA private key** – Store in HSM or secure vault, never in version control
- 📅 **Monitor expiration** – Set up alerts for certificates expiring within 30 days

### Security Best Practices

- ⚠️ **Not for public-facing applications** – mTLS requires all clients to have certificates
- 🔒 **Restrict namespace access** – Limit RBAC permissions to the ingress namespace
- 📋 **Maintain certificate inventory** – Track issued certificates and expiration dates
- 🔄 **Implement revocation** – Use CRL or OCSP for compromised certificates

### DNS and Networking

- 🌐 **Configure DNS A records** – Point your domain to the Ingress Controller's LoadBalancer IP
- 🔍 **Enable detailed logging** – Log client certificate subjects for audit trails
- 📊 **Monitor failed authentications** – Set up alerts for unusual 400 error rates

---

## Troubleshooting

### Issue: 400 Bad Request – No Client Certificate

**Symptoms:**
```bash
curl -k https://your-domain.com
# Returns: 400 Bad Request
```

**Solutions:**
1. Verify client certificate is being sent: `curl --cert client.crt --key client.key`
2. Check CA secret exists in the correct namespace
3. Verify `auth-tls-verify-client: "on"` annotation is present
4. Check NGINX controller logs: `kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx`

### Issue: SSL Certificate Verify Failed

**Symptoms:**
```bash
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

**Solutions:**
1. Import CA certificate: `curl --cacert ca.crt`
2. Verify server certificate chain is complete
3. Check SAN matches your domain

### Issue: Client Certificate Not Passed to Backend

**Symptoms:** Backend doesn't receive client certificate headers

**Solutions:**
1. Enable `auth-tls-pass-certificate-to-upstream: "true"` annotation
2. Verify backend reads `X-SSL-Client-Cert` header
3. Check NGINX configuration for certificate forwarding

---

## Conclusion

Implementing **per-ingress mTLS** with NGINX Ingress Controller provides flexible, fine-grained security for your Kubernetes applications. This approach is ideal when you need:

- ✅ **Selective mTLS enforcement** – Only specific services require client certificates
- ✅ **Mixed public/internal services** – Some endpoints public, others protected
- ✅ **Gradual migration** – Enable mTLS incrementally across your applications

By requiring client certificates at the ingress boundary for selected services, you protect sensitive applications while maintaining accessibility for public-facing endpoints.

---

**Found this guide helpful?** Share your mTLS implementation experiences in the comments below.

