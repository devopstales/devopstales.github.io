---
title: Move Windows Certificate Authority to another server
url: https://devopstales.github.io/windows/move-certificate-authority-to-anoter-server/
date: 2024-10-22
---


In this post I will show how to move Windows Certificate Authority role to another Server.

<!--more-->

### Backup the current Root CA

Open the Certification Authority manager.

![Example image](/img/include/move-root-ca01.webp)

Right click the name of the CA and select `All Tasks > Back up CA`.

![Example image](/img/include/move-root-ca02.webp)

The `Certification Authority Backup Wizard` opens. Click `Next`. 

Select both `Private key and CA certificate` and `Certificate database and certificate database log` options. Click `Browse` and select a backup location then click `Next`.

![Example image](/img/include/move-root-ca03.webp)

Enter a `Password` to gain access to the private key and click `Next`.

![Example image](/img/include/move-root-ca04.webp)

### Backup the CA registry key

Now run the `regedit` command to export the registry key.

Navigate to `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration`, right click the `Root CA name` and select `Export`.

![Example image](/img/include/move-root-ca05.webp)

Select the path to store the file, specify the `File name` and click `Save`.

![Example image](/img/include/move-root-ca06.webp)

### Remove the CA role

The CA role must be removed from the server to dismiss.
From the `Server Manager` select `Manage > Remove Roles and Features` option and Click `Next`.

![Example image](/img/include/move-root-ca07.webp)

Choose `Select a server from the server pool` and click `Next`. Untick the `Certification Authority` role and `Next`.

![Example image](/img/include/move-root-ca08.webp)

Click `Remove Features`.

The `Certification Authority` role has been removed from the current server. Click `Next`.
Select `Restart the destination server automatically if required` and click `Remove`.

### Install the CA role on the new server

In the new server, open the `Server Manager` and click `Add roles and features`.

![Example image](/img/include/move-root-ca10.webp)

Select `Role-based or feature-based installation` option and click `Next`.

![Example image](/img/include/move-root-ca11.webp)

Choose `Select a server from the server pool`, select the server and click `Next`. Then select the `Active Directory Certificate Services` role and Click `Add Features` when prompted. Then click Next to install the role.

![Example image](/img/include/move-root-ca12.webp)

Click `Next` to install selected services.

![Example image](/img/include/move-root-ca14.webp)

Accept default role services and click `Next`. Select `Restart the destination server automatically if required` and click `Install`.

When the installation process starts, you can click `Close`.

### Configure the new CA

When the installation procedure completes, from the `Server Manager` click the yellow exclamation mark and click on the link `Configure Active Directory Certificate Services on the destination server`.

![Example image](/img/include/move-root-ca15.webp)

Make sure to use an account with`Enterprise Administrator` permissions. Click `Next`.

![Example image](/img/include/move-root-ca16.webp)

Select the two role services and click `Next`.

![Example image](/img/include/move-root-ca17.webp)

Select `Enterprise CA` as CA type and click `Next`.

![Example image](/img/include/move-root-ca18.webp)

Select `Use existing private key` and choose `Select a certificate and use its associated private key`. Click `Next`.

![Example image](/img/include/move-root-ca19.webp)

Click `Import`. Click `Browse` and select the certificate exported from the old CA and enter the `Password`. Click `OK`.

Select the imported certificate and click `Next`. Leave default locations and click `Next`.

Click `Close` when the configuration completes successfully.

### Import the registry key

Last step is the import of the registry key previously exported from the old CA.

Before importing the registry key we need to `change the name server` with the new one. Right click the registry key file (ca_config.reg in the example) and select `Edit`.

![Example image](/img/include/move-root-ca20.webp)

Locate the `CAServerName` entry and change the name with the current server name and save the file.

Now open the Command Prompt and stop the ca service with the command:

```bash
net stop certsvc
```

Double click on the registry file to import the settings. Click `Yes` to confirm the import.

![Example image](/img/include/move-root-ca21.webp)

Click `OK` when values have been added successfully.

### Restore the database

Open the `Certification Authority manager` and right click the CA name and select` All Taks > Restore CA`.

![Example image](/img/include/move-root-ca22.webp)

The `Certification Authority Restore Wizard` opens. Click `Next`.

![Example image](/img/include/move-root-ca23.webp)

Select both `Private key and CA certificate` and `Certificate database and certificate database log` options. Click `Browse` and select the location where the database is located then click `Next`.

![Example image](/img/include/move-root-ca24.webp)

Enter the `Password` to gain access to the private key and click `Next` then click `Finish` to restore the database.

Click `Yes` on the pop-up window to start Active Directory Certificate Services.

### What About Certificate Templates? Do I need to Move Them?

`No`! Certificate templates are actually stored in `Active Directory`, NOT in/on the actual Certificate Services server, (that’s why sometimes they take a while to appear after you create them!)

![Example image](/img/include/move-root-ca25.webp)

