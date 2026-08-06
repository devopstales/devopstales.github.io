---
title: Nextcloud SSO
url: https://devopstales.github.io/sso/nextcloud-sso/
date: 2019-05-25
keywords: Keycloak, oauth2, SSO, OIDC, NextCloud
---


Nextcloud is a suite of client-server software for creating and using file hosting services. Nextcloud application functionally is similar to Dropbox.

<!--more-->

## Configuring Keycloak and Nextcloud
### Keycloak side
- login to keycloak using the admin account
- Under `Clients`, create a new client with `Client ID` "nextcloud" and `Root URL` "cloud.devopstales.intra"
- On next screen, under the `Settings` tab, change `Access Type` from `public` to `confidential`, then Save
- Go the the `Credentials` tab, note the `Secret`
- OPTIONAL: If there is no registered user yet you can create a test user: go to `Users`, click the `Add User` button, fill the `Username` with "test" and save. Then go to the `Credentials` tab, put the new password, toggle the `Temporary` option to `OFF`, press `Reset Password` and confirm

Keycloak is now ready to be used for Nextcloud.

### NextCloud side
- login to your Nextcloud instance with the admin account
- Click on the user profile, then `Apps`
![Example image](/img/include/nextcloud_sso1.webp)<br><br>
- Go to `Social & communication` and install the `Social Login` app
- Go to `Settings` (in your user profile) the `Social Login`<br><br>
![Example image](/img/include/nextcloud_sso2.webp)<br><br>
- Add a new `Custom OpenID Connect` by clicking on the `+` to its side
- Fill the following:
  - `Title` -> "keycloak"
  - `Authorize url` -> `https://keycloak.devopstales.intra:8443/auth/realms/mydomain/protocol/openid-connect/auth`
  - `Token url` -> `https:/keycloak.devopstales.intra:8443/auth/realms/mydomain/protocol/openid-connect/token`
  - `Client id` -> "nextcloud"
  - `Client Secret` -> put the secret you noted down during the Keycloak configuration
  - `Scope` -> "openid"
- Press `Save`

Your Nextcloud instance is now configured. Log out and log back in using the `Alternative Logins -> keycloak` method on the login page. It should redirect you to a keycloak auth form where you can log in with a registered keycloak user, then back to Nextcloud where you are now logged.
![Example image](/img/include/nextcloud_sso3.webp)<br><br>

