---
title: Free sso for Mattermost Teams Edition
url: https://devopstales.github.io/sso/mattermost-keycloak-sso/
date: 2020-02-16
keywords: Keycloak, oauth2, SSO, OIDC, gitlab mattermost, mattermost vs slack, mattermost client
---


In this article I will show you how to use Keycloak as an authentication provider for  Mattermost Teams Edition.

<!--more-->

Slack and Mattermost are very similar products. With Mattermost Teams Edition you can get almost all the functionality of the payed Slack. The only problem is sso. In Mattermost you can use google sso login in the E20 licensing what is more costly the Slack. (Slack: 8$/user Mattermost E20: 8.5$/user) In  Mattermost Teams Edition (the free edition of Mattermost) the only authentication provider you can use is gitlab, but thanks to [wadahiro](https://qiita.com/wadahiro/items/8b118c34aae904353865) we can use Keycloak instead of gitlab.


### Configurate MAttermost
```
nano /etc/mattermost/config.json
...
"GitLabSettings": {
    "Enable": false,
    "Secret": "<secret>",
    "Id": "mattermost",
    "Scope": "",
    "AuthEndpoint": "https://keycloak.mydomain.intra/auth/realms/mydomain/protocol/openid-connect/auth",
    "TokenEndpoint": "https://keycloak.mydomain.intra/auth/realms/mydomain/protocol/openid-connect/token",
    "UserApiEndpoint": "https://keycloak.mydomain.intra/auth/realms/mydomain/protocol/openid-connect/userinfo"
},
```

### Create Client on Keycloak

In keycloak go to `Configure > Clients` and create a new client for Mattermostw With the fallowing data:

* Standard Flow Enabled: `ON`
* Access Type: `confidential`
* Valid Redirect URIs: `http://<Mattermost-FQDN>/signup/gitlab/complete`

![image](/img/include/mattermost_sso1.webp) <br>

### Create mapper for correct data
Mattermost [want](https://github.com/mattermost/mattermost-server/blob/v4.10.0/model/gitlab/gitlab.go#L19-L25) the fallowing data from the authentication provider:

```
type GitLabUser struct {
	Id       int64  `json:"id"`
	Username string `json:"username"`
	Login    string `json:"login"`
	Email    string `json:"email"`
	Name     string `json:"name"`
}
```

Create mapping for username:

![image](/img/include/mattermost_sso2.webp) <br>

The only problematic data is the ID. For the test run you can use a self generated id like this:

![image](/img/include/mattermost_sso3.webp) <br>

### Create user for test
For testing purposes we’re going to create a local user in Keycloak.

![image](/img/include/mattermost_sso4.webp) <br>

Now we need to create an attribute for this user with the mattermost id and the value should be an integer between 1 and 9999999999999999999.

![image](/img/include/mattermost_sso5.webp) <br>

If you have many users you didn't want to create id for them manually. If you use LDAP for users the best fit for this need was the employeeNumber or EmployeeId ldap attribute.

![image](/img/include/mattermost_sso6.webp) <br>

If you use ActiveDirectory this attribute maybe null so we need to generate it with a powershell script.
```
ForEach ($User in ((Get-ADUser -Filter * -Properties SamAccountName,EmployeeId)))
{
if ( ([string]::IsNullOrEmpty($User.EmployeeId)))
{
$DATE = (Get-ADuser $User.SamAccountName -Properties whencreated).whencreated.ToString('yyMMddHHmmss')
$RANDOM = (Get-Random -Maximum 9999)
$DATA = -join ($DATE, $RANDOM)
$User.SamAccountName
$DATA
Set-ADUser $User.SamAccountName -employeeID $DATA
}
}
```

