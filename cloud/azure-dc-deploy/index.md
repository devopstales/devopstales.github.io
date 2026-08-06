---
title: How to deploy a Domain Controller on Microsoft Azure
url: https://devopstales.github.io/cloud/azure-dc-deploy/
date: 2021-12-07
keywords: Azure, DC, Domain Controller, hybrid
---


In this pos I will show you how you can create a hybrid Acrive Directory Domain with on-premiss and Azure DCs.
<!--more-->

### Requirements

* An Azure AD tenant with an active subscription.
* A Virtual Network in Azure that doesn’t overlap with your on-premises network.
* A continuous line of sight between your on-premises domain controller and Microsoft Azure (Azure VPN Gateway, ExpressRoute or an NVA). 

### Deploy A Virtual Machine

- Go to the azure portal (https://portal.azure.com) and login
- Create a new Windows Server resource. I Recommened using Windows Server 2019. 

![Bacic info](/img/include/azuredc01.webp)

For safety reasons, you should set `allow selected ports` to `none`.

![disallow selected ports](/img/include/azuredc02.webp)

- Click Next to configure vm disks.

> A Single VM without premium SSD’s has an SLA of 99.95%. A Single VM with premium SSD’s (all disks) has an SLA of 99.99%. I Recommend using premium disks for your domain controller.

- Add a second (premium ssd) disk with host caching set to none. This disk will contain the database, logs and sysvol folders.

![add premium ssd](/img/include/azuredc03.webp)

- Click Next to configure networking. Attach the VM to your existing vNet that’s connected with your on-premises domain. Don’t assign a public IP address to your virtual machine as recommended by Microsoft – use a VPN or Azure Bastion to connect to the machine. 

![configure network](/img/include/azuredc04.webp)

- Finish all steps to create the virtual machine. Don’t enable `Login with AAD credentials` or `Auto-shutdown`.

### Configure static IP

The virtual machine must have a static IP address.

- Select network interface of your new virtual machine

![static ip](/img/include/azuredc05.webp)

![static ip](/img/include/azuredc06.webp)

- Select Static and configure the IP address. Don’t forget to click save – a reboot may be required. You should never configure the static IP address on the VM itself as you do on-premises.

![static ip](/img/include/azuredc07.webp)

### Domain join

- Test if you can ping the VM from your on-premises domain controller and the other way around.
- Open Active Directory Sites & Services on your on-premises domain controller.
- Create a new site

![new site](/img/include/azuredc08.webp)

![new site](/img/include/azuredc09.webp)

- Right click Subnets and select New Subnet.

![new subnet](/img/include/azuredc10.webp)

![new subnet](/img/include/azuredc11.webp)

- Start Add Roles and Features on the Azure VM.
- Add the Active Directory Domain Services role and all necessary features.
- Promote this server to a domain controller.
- Select Add a domain controller to an existing domain.

![join domain](/img/include/azuredc12.webp)

![join domain](/img/include/azuredc13.webp)

![join domain](/img/include/azuredc14.webp)

- Reboot the virtual machine.

### Validate DC DNS Settings on Azure

When the virtual machine is back online, it probably has static DNS servers configured – this happened because of the AD DC roles. Change this back to Obtain DNS server address automatically.

![configure dns server](/img/include/azuredc15.webp)

![configure dns server](/img/include/azuredc16.webp)

