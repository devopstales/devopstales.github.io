---
title: Install s3cmd with CEHP Radosgateway
url: https://devopstales.github.io/linux/s3cmd-with-radosgw/
date: 2019-08-04
---


s3cmd is a cli utility for s3.

<!--more-->

```
root@pve1:~# apt-get install s3cmd
root@pve1:~# s3cmd --configure
Access Key: xxxxxxxxxxxxxxxxxxxxxx
Secret Key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

root@pve1:~#  s3cmd mb s3://devopstales
Bucket 's3://devopstales/' created
```


### Create User
```
adosgw-admin user create --uid={username} --display-name="{display-name}" \[--email={email}\]
radosgw-admin user create --uid=devopstales --display-name="devopstales" --email=devopstales@devopstales.intra
```

### User Info
```
radosgw-admin user info --uid=devopstales
```

### Modify User
```
radosgw-admin user modify --uid=devopstales --display-name="John Doe"
```

### Disable and Enable User
```
radosgw-admin user suspend --uid=devopstales
radosgw-admin user enable --uid=devopstales
```

### Remove User
```
radosgw-admin user rm --uid=devopstales
```

# Add quota for User
```
radosgw-admin quota set --uid="devopstales@devopstales.intra" --quota-scope=bucket --max-size=30G
radosgw-admin quota enable --quota-scope=bucket --uid="devopstales@devopstales.intra"
```

### Set Bucket Policy
Create a Bucket Policy fot devopstales@devopstales.intra to allow privileges for devopstales Bucket.

```
cat <EOF> bucket-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": ["arn:aws:iam:::user/devopstales2@devopstales.intra"]},
    "Action": ["s3:ListBucket", "s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
    "Resource": [
      "arn:aws:s3:::devopstales",
      "arn:aws:s3:::devopstales/*"
    ]
  }]
}
EOF

s3cmd setpolicy ./bucket-policy.json s3://devopstales
```

