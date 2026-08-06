---
title: How to create your first Chef Cookbook
url: https://devopstales.github.io/linux/chef-first-cookbook/
date: 2020-05-07
keywords: chef-server, configuration management
---

I this post I will show you how you can create a basic chef structure.
<!--more-->

### Create environments
```
cd ~/chef-repo

mkdir environments
nano environments/production.json
{
  "name": "Production",
  "description": "",
  "cookbook_versions": {

  },
  "json_class": "Chef::Environment",
  "chef_type": "environment",
  "default_attributes": {

  },
  "override_attributes": {

  }
}

nano environments/development.json
{
  "name": "Development",
  "description": "",
  "cookbook_versions": {

  },
  "json_class": "Chef::Environment",
  "chef_type": "environment",
  "default_attributes": {

  },
  "override_attributes": {

  }
}
```

```
knife environment from file environments/production.json
knife environment from file environments/development.json
```

### Create roles
```
mkdir roles
nano roles/base.json
{

  "name": "base",
  "description": "Default operation role",
  "json_class": "Chef::Role",

  "override_attributes": {
    "chef_client": {
      "config": {
        "interval": 900,
        "splay": 30
      }
    }
  },

  "chef_type": "role",

  "run_list": [
    "recipe[operation]"
  ],

  "env_run_lists": {
  }

}
```

```
knife role from file roles/base.json
```

### Create node config
```
mkdir nodes
nano nodes/test.mydomain.intra.json
{
  "name": "test.mydomain.intra",
  "chef_environment": "Production",
  "normal": {
  	"tags": [

    ]
  },
  "run_list": [
	"role[base]"
]

}
```

```
knife node from file nodes/test.mydomain.intra.json
```

### Create cookbook
```
cd chef-repo/cookbooks
chef generate cookbook operation
nano operation/recipes/default.rb
include_recipe 'operation::packages'

nano operation/recipes/packages.rb
package "nano" do
  action :install
end
```

```
cd ..
knife cookbook upload operation
knife cookbook list
```

### Test cookbook
```
knife ssh 'name:test.mydomain.intra' 'sudo chef-client' -x root
```

