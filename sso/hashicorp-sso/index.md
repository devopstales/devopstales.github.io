---
title: SSO for hashicorp vault
url: https://devopstales.github.io/sso/hashicorp-sso/
date: 2019-05-20
keywords: Keycloak, HashiCorp Vault, oauth2, SSO, OIDC
---


In this post I wil shiw you hiw to configure Hashicorp vault with Keycloak for SSO.
<!--more-->

```
vault auth enable oidc

vault write auth/oidc/config \
    oidc_discovery_url="https://sso.devopstales.intra/auth/realms/mydomain" \
    oidc_client_id="web" \
    oidc_client_secret="07d66ebd-1018-46c6-9c88-80aa3d4c2f68" \
    default_role="reader"
```

```
vault write auth/oidc/role/reader \
        bound_audiences="web" \
        allowed_redirect_uris="http://192.168.0.112:8200/ui/vault/auth/oidc/oidc/callback" \
        allowed_redirect_uris="http://192.168.0.112:8250/oidc/callback" \
        user_claim="sub" \
        policies="reader"
```

```
nano reader.hcl
# Read permission on the k/v secrets
path "/secret/*" {
    capabilities = ["read", "list"]
}

nano manager.hcl
# Manage k/v secrets
path "/secret/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
}
```

```
vault policy write reader reader.hcl
vault policy write manager manager.hcl

vault policy list
```

![Example image](/img/include/vault-oidc.webp)

