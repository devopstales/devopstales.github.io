---
title: OpenProject SSO
url: https://devopstales.github.io/sso/openproject-sso/
date: 2019-04-06
keywords: Keycloak, oauth2, SSO, OIDC, openproject
---


Configurate openproject to use Keycloak as sso Identity Provider.

<!--more-->

### Configure OpenProject
Login to openproject with admin and change the config of Self-registrtion to automatic account activation:
Administraion > System Settings > Authentication > Self-registration

```
nano /opt/openproject/config/configuration.yml
default:
  omniauth_direct_login_provider: openid
  openid_connect:
    openid:
      host: "sso.devopstales.intra"
      identifier: "project"
      secret: "57583084-b54b-4b32-935b-73776f27b89f"
      icon: "openid_connect/auth_provider-google.webp"
      display_name: "SSO"
      authorization_endpoint: "http://sso.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/auth"
      token_endpoint: 'http://sso.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/token'
      userinfo_endpoint: 'http://sso.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/userinfo'
      end_session_endpoint: 'http://sso.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/logout'
      check_session_iframe: 'http://sso.devopstales.intra/auth/realms/mydomain/protocol/openid-connect/login-status-iframe.html'
      sso: true
      issuer: 'http://project.devopstales.intra/login'
      discovery: false

service openproject restart
```

