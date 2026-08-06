---
title: SSO login to Gitlab
url: https://devopstales.github.io/sso/gitlab-keycloak/
date: 2019-04-06
keywords: Keycloak, Gitlab, oauth2, SSO, OIDC
---


Configurate Gitab to use Keycloak as SSO Identity Proider.

<!--more-->

### Configurate Keycloak
Login to Keycloak and create client for Gitlab:
![Example image](/img/include/gitlab-keycloak1.webp)

At Mappers create mappers for all user information to GitLab:

* Name: name
  * Mapper Type: User Property
  * Property: Username
* Name: email
  * Mapper Type: User Property
  * Property: Email
* Name: first_name
  * Mapper Type: User Property
  * Property: FirstName
* Name: last_name
  * Mapper Type: User Property
  * Property: LastName

### Configurate Gitlab
```
nano /etc/gitlab/gitlab.rb
gitlab_rails['omniauth_enabled'] = true
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_allow_single_sign_on'] = ['oauth2_generic']
# gitlab_rails['omniauth_auto_sign_in_with_provider'] = 'oauth2_generic'

gitlab_rails['omniauth_providers'] = [
{
        'name' => 'oauth2_generic',
        'app_id' => 'gitlab',
        'app_secret' => 'KEYCLOAK SECRET GOES HERE',
        'args' => {
        client_options: {
                'site' => 'http://sso.devopstales.intra', # including port if necessary
                'user_info_url' => '/auth/realms/devopstales/protocol/openid-connect/userinfo',
                'authorize_url' => '/auth/realms/devopstales/protocol/openid-connect/auth',
                'token_url' => '/auth/realms/devopstales/protocol/openid-connect/token',
        },
        user_response_structure: {
        #root_path: ['user'], # i.e. if attributes are returned in JsonAPI format (in a 'user' node nested under a 'data' node)
        attributes: { email:'email', first_name:'given_name', last_name:'family_name', name:'name', nickname:'preferred_username' }, # if the nickname attribute of a user is called 'username'
        id_path: 'preferred_username'
        },
        }
}
]

gitlab-ctl reconfigure
```

### Gitlab Mattermost config
```
# on gitlab gui:
login: admin area / Applications / new
Redirect URI use:
http://mattermost.devopstales.intra/login/gitlab/complete
http://mattermost.devopstales.intra/signup/gitlab/complete

# configfile
nano /etc/gitlab/gitlab.rb

mattermost_external_url 'http://mattermost.devopstales.intra'
mattermost['enable'] = true
mattermost['service_address'] = "127.0.0.1"
mattermost['service_port'] = "8065"
mattermost['sql_driver_name'] = 'postgres'
mattermost['sql_data_source'] = "postgres://mmuser:Password1@127.0.0.1:5432/mattermost?sslmode=disable&connect_timeout=10"
mattermost['log_file_directory'] = '/var/log/gitlab/mattermost/'
mattermost_nginx['enable'] = false

mattermost['gitlab_enable'] = true
mattermost['gitlab_id'] = "<ID>" # oauth id drom gitlab gui
mattermost['gitlab_secret'] = "<token>" # oauth token drom gitlab gui
mattermost['gitlab_scope'] = ""
mattermost['gitlab_auth_endpoint'] = "http://gitlab.devopstales.intra/oauth/authorize"
mattermost['gitlab_token_endpoint'] = "http://gitlab.devopstales.intra/oauth/token"
mattermost['gitlab_user_api_endpoint'] = "http://gitlab.devopstales.intra/api/v4/user"

gitlab-ctl reconfigure
```

