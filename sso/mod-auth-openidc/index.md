---
title: Apaceh2 oauth plugin
url: https://devopstales.github.io/sso/mod-auth-openidc/
date: 2019-05-13
keywords: Keycloak, oauth2, apache2, httpd, SSO, OIDC
---


Configure Apache plugin to use Keycloak as a user backend for login with OpenID and SSO.

<!--more-->
mod_auth_openidc is an OpenID Connect Relying Party implementation for Apache HTTP Server 2.x

### Install the plugin
```
yum install mod_auth_openidc httpd php mod_ssl -y

mkdir -p /var/www/html/oauth/protected
echo "index" > /var/www/html/oauth/index.htm
```

```
nano /var/www/html/oauth/protected/index.php
<!DOCTYPE html>
<html lang="en">

<head>

   <meta charset="utf-8">
   <meta http-equiv="X-UA-Compatible" content="IE=edge">
   <meta name="viewport" content="width=device-width, initial-scale=1">
   <meta name="description" content="">
   <meta name="author" content="">

   <title>OpenID Connect: Received Claims</title>

</head>

<body>
         <h3>
            Claims sent back from OpenID Connect via the Apache module
         </h3>
         <br/>


   <!-- OpenAthens attribtues -->
      <?php session_start(); ?>

         <h2>Claims</h2>
         <br/>
         <div class="row">

               <table class="table" style="width:80%;" border="1">
                 <?php foreach ($_SERVER as $key=>$value): ?>
                    <?php if ( preg_match("/OIDC_/i", $key) ): ?>
                       <tr>
                          <td data-toggle="tooltip" title=<?php echo $key; ?>><?php echo $key; ?></td>
                          <td data-toggle="tooltip" title=<?php echo $value; ?>><?php echo $value; ?></td>
                       </tr>
                    <?php endif; ?>
                 <?php endforeach; ?>
               </table>

</body></html>
```

### Create vhost
```
nano /etc/httpd/conf.d/aouth-site.conf
# NameVirtualHost *:80
<VirtualHost *:80>
   ServerName oauth.devopstales.intra
   DocumentRoot /var/www/oauth/
   Redirect permanent / https://oauth.devopstales.intra
</VirtualHost>

<VirtualHost *:443>
    ServerAdmin webmaster@example.com
    ServerName oauth.devopstales.intra
    ServerAlias www.oauth.devopstales.intra
    DocumentRoot /var/www/html/oauth/
    DirectoryIndex index.html index.php
    ErrorLog /var/log/httpd/oauth-error.log
    CustomLog /var/log/httpd/oauth-access.log combined

    SSLEngine on
    SSLCertificateFile /etc/httpd/ssl/domain.pem
    SSLCertificateKeyFile /etc/httpd/ssl/domain.pem
    SSLCertificateChainFile /etc/httpd/ssl/domain.pem

    # keycloak server
    OIDCProviderMetadataURL http://sso.devopstales.intra/auth/realms/mydomain/.well-known/openid-configuration
    # for self signed certificate
    OIDCSSLValidateServer Off
    OIDCClientID web
    OIDCClientSecret 5b721a2b-681f-402d-807c-b98c80672c16
    OIDCRedirectURI http://oauth.devopstales.intra/protected/redirect_uri
    OIDCCryptoPassphrase passphrase
    OIDCJWKSRefreshInterval 3600

    <Location /protected/>
       AuthType openid-connect
       Require valid-user
    </Location>

</VirtualHost>
```

### Start apache
```
systemct start httpd
```

