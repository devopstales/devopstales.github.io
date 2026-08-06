---
title: Nagios Cross Platform Agent
url: https://devopstales.github.io/monitoring/nagios-ncpa/
date: 2019-05-07
keywords: nagios, ncpa
---


In this article I will show you how to add Remote Linux machine and it’s services to Nagios Monitoring host using NCPA agent.

<!--more-->

### What is NCPA

NCPA is written in Python and is able to run on almost any Operating System. IT build official binaries for Windows, Mac OS X, and various Linux flavors.

### NCPA Client

```
rpm -Uvh https://repo.nagios.com/nagios/7/nagios-repo-7-3.el7.noarch.rpm
yum install ncpa -y

nano /usr/local/ncpa/etc/ncpa.cfg
# [listener]
# allowed_hosts = <nagios host>
[api]
community_string = Password1
[nrdp]
# hostname =
# [nrdp]
# parent =
# token =
[plugin directives]
plugin_path = /usr/lib64/nagios/plugins/

systemctl enable ncpa_listener
systemctl start ncpa_listener
systemctl status ncpa_listener

# https://192.168.0.100:5693/
```

### NCPA Server

```
cd /tmp
wget https://assets.nagios.com/downloads/ncpa/check_ncpa.tar.gz
tar xvf check_ncpa.tar.gz
chown nagios:nagios check_ncpa.py
chmod 775 check_ncpa.py
mv check_ncpa.py /usr/lib64/nagios/plugins
```


Test the commands
```
/usr/lib64/naemon/plugins/check_ncpa.py -H localhost -t 'Password1' -P 5693 -M system/agent_version
/usr/lib64/naemon/plugins/check_ncpa.py -H localhost -t 'Password1' -P 5693 -M cpu/percent -w 20 -c 40 -q 'aggregate=avg'
/usr/lib64/naemon/plugins/check_ncpa.py -H localhost -t 'Password1' -P 5693 -M memory/virtual -w 50 -c 80 -u G
/usr/lib64/naemon/plugins/check_ncpa.py -H localhost -t 'Password1' -P 5693 -M processes -w 150 -c 200

# Run custom plugin trouth NCPA
/usr/lib64/naemon/plugins/check_ncpa.py -H localhost -t 'Password1' -P 5693 -M plugins/check_users -q args="-w 5 -c 10"
```

```
# store password in macro
nano /etc/nagios/resource.cfg
USER10 = Password1

# createcustom commands for ncpa
nano /etc/nagios/commands.cfg
define command {
    command_name    check_ncpa
    command_line    $USER1$/check_ncpa.py -H $HOSTADDRESS$ -t $USER10$ -P 5693 $ARG1$
}

define command {
    command_name    check_ncpa_cpu
    command_line    $USER1$/check_ncpa.py -H $HOSTADDRESS$ -t $USER10$ -P 5693 cpu/percent -w 20 -c 40 -q 'aggregate=avg'
}
```

```
nano /etc/nagios/conf.d/ncpa-test.cfg
        define host{
        use                    generic-host
        host_name              devopstales
        address                192.168.0.20
}

define service{
        use                     generic-service
        host_name               devopstales
        service_description     NCPA Version
        check_command           check_ncpa|system/agent_version
        }

define service{
        use                     generic-service
        host_name               devopstales
        service_description     CPU Load
        check_command           check_ncpa_cpu
        }
```

### Restart nagios
```
nagios -v /etc/nagios/nagios.cfg

Total Warnings: 0
Total Errors:   0

service nagios restart
```

