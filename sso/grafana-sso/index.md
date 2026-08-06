---
title: SSO login to Grafana
url: https://devopstales.github.io/sso/grafana-sso/
date: 2019-06-28
keywords: Keycloak, grafana, oauth2, SSO, OIDC
---


Configurate Gitab to use Keycloak as SSO Identity Proider.

<!--more-->

### Configurate Keycloak
Login to Keycloak and create client for Grafana:
![Example image](/img/include/gitlab-keycloak1.webp)

### Configurate Gitlab
```
nano /etc/grafana/grafana.ini
#################################### Generic OAuth ##########################
[auth.generic_oauth]
enabled = true
name = SSO
allow_sign_up = true
client_id = gitlab
client_secret = 47fd3013-4333-4825-bbfa-b7688548d9cf
# for old version
# scopes = user:email,read:org
scopes = openid email profile
auth_url = https://sso.devopstales.intra/auth/realms/gitlab/protocol/openid-connect/auth
token_url = https://sso.devopstales.intra/auth/realms/gitlab/protocol/openid-connect/token
api_url = https://sso.devopstales.intra/auth/realms/gitlab/protocol/openid-connect/userinfo
;team_ids =
;allowed_organizations =

```

