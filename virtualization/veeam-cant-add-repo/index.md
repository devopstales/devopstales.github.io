---
title: veeam backup: Cant Add Repository to a Scale Out Backup Repository?
url: https://devopstales.github.io/virtualization/veeam-cant-add-repo/
date: 2022-07-22
keywords: esx, esxi, vmware, vSphere, veeam backup, Unable to add extent, because it serves as the target for one or more job types which are not supported by a scale-out backup repository
---


When adding repositories to a Veeam Scale Out Backup Repository you may see this error:  Cant Add Repository to a Scale Out Backup Repository? In this Pos I will show you how you can fix is issue.

<!--more-->

### The Issue

When I try to add the defult repo in veeam bakup to the Scale Out Backup Repository I get the fallowin error:

> Unable to add extent {Repository-Name} because it serves as the target for one or more job types which are not supported by a scale-out backup repository.

Selecting 'Show jobs' shows:

![Show jobs](/img/include/veeam-001.webp)

### The Solution

Veeam Backup create a "Job" to vackup its internal database in case you you wanted to reinstall or migrate Veeam to another server. It gets created automatically, to the default repository. 
To Fix: Simply create a new repository just for the Configuration-Backup-Job. I typically name the repository 'Config-Backup-Repository' to avoid confusion in the future.

![Repoosirory List](/img/include/veeam-002.webp)

Now simply change the job to use the new repository, this is NOT done where all the other jobs are configured!
Select 'Options > Configuration Backup > Change the Repository'  (I manually run it, by clicking Backup-Now) at this point, just to make sure all is well.

![Change Job](/img/include/veeam-003.webp)

You should now be able to create your Scale Out Backup Repository without an error.

