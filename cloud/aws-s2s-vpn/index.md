---
title: AWS - pfsense: Site-to-site VPN using static routes
url: https://devopstales.github.io/cloud/aws-s2s-vpn/
date: 2022-02-22
keywords: AWS, vpn, ipsec, pfsense
---


In this post I willll show you how to configure a VPN between pfSense and AWS using static routes. 
<!--more-->

To create a VPN on AWS side you need the following Components:

* Customer Gateway - This is represent the on-premise side of the vpn
* virtual private gateway - this is a router in the aws
* vpn Connection
* virtual priveta cloud

`vpc -> virtual private gateway -> vpn Connection -> Customer Gateway`

![vpn infra](/img/include/vpn-how-it-works-vgw.webp)

We need to create this components and connect them to each other.

### Customer Gateway

```bash
# set to your own public ip
export CLIENT_PUBLIC_IP=1.2.3.4

# Create the customer gateway using the following AWS command:
aws ec2 create-customer-gateway --type ipsec.1 --public-ip $CLIENT_PUBLIC_IP

{
    "CustomerGateway": {
        "CustomerGatewayId": "cgw-0e11f167",
        "IpAddress": "1.2.3.4",
        "State": "available",
        "Type": "ipsec.1",
        "BgpAsn": "65000"
    }
}

export CUSTOMER_GATEWAY=cgw-0e11f167
```

### Create a Virtual Private Gateway

Create a target gateway and attach it to your VPC network.

```bash
# Create a virtual private gateway with a specific AWS-side ASN:
aws ec2 create-vpn-gateway --type ipsec.1

{
    "VpnGateway": {
        "AmazonSideAsn": 64512,
        "State": "available",
        "Type": "ipsec.1",
        "VpnGatewayId": "vgw-9a4cacf3",
        "VpcAttachments": []
    }
}

export VPN_GATEWAY_ID=vgw-9a4cacf3
export VPC_ID=

# Attach the virtual private gateway to your VPC network:
aws ec2 attach-vpn-gateway --vpn-gateway-id $VPN_GATEWAY_ID --vpc-id $VPC_ID
```

### Create a VPN Connection

```bash
export AWS_TIP=169.254.0.0/30
# random string for secret
export SHARED_SECRET=g23r8gr7grg23r8g2fnmf
# my network on the on-premise side
export ONPREM_NETWORK=192.168.1.0/24

aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id $CUSTOMER_GATEWAY \
    --vpn-gateway-id $VPN_GATEWAY_ID \
    --options TunnelOptions="[{TunnelInsideCidr=$AWS_TIP,PreSharedKey=$SHARED_SECRET}]",StaticRoutesOnly=true,LocalIpv4NetworkCidr=$ONPREM_NETWORK

{
    "VpnConnection": {
        "CustomerGatewayConfiguration": "...configuration information...",
        "CustomerGatewayId": "cgw-0e11f167",
        "Category": "VPN",
        "State": "pending",
        "VpnConnectionId": "vpn-123123123123abcab",
        "VpnGatewayId": "vgw-9a4cacf3",
        "Options": {
            "EnableAcceleration": false,
            "StaticRoutesOnly": true,
            "LocalIpv4NetworkCidr": "192.168.1.0/24",
            "RemoteIpv4NetworkCidr": "0.0.0.0/0",
            "TunnelInsideIpVersion": "ipv4",
            "TunnelOptions": [
                {
                    "OutsideIpAddress": "203.0.113.3",
                    "TunnelInsideCidr": "169.254.0.0/30",
                    "PreSharedKey": "g23r8gr7grg23r8g2fnmf"
                },
                {}
            ]
        },
        "Routes": [],
        "Tags": []
    }
}
```

In the `TunnelOptions` you can configure other options of the vpn like:

```bash
IKEVersions=[{Value=ikev2})
Phase2DHGroupNumbers=[{Value=15})
Phase1DHGroupNumbers=[{Value=15})
Phase2IntegrityAlgorithms=[{Value=SHA2-256})
Phase1IntegrityAlgorithms=[{Value=SHA2-256})
Phase2EncryptionAlgorithms=[{Value=AES256-GCM-16})
Phase1EncryptionAlgorithms=[{Value=AES256-GCM-16})
```

### Configure Routing

```bash
aws ec2 create-route --route-table-id rtb-89012345678901234 \
--destination-cidr-block 172.31.0.0/16 \
--transit-gateway-id tgw-56789012345678901 
```

### Download the configuration file
After you create the Site-to-Site VPN connection, you can download a sample configuration file to use for configuring the customer gateway device.

> The configuration file is an example only and might not match your intended Site-to-Site VPN connection settings entirely. It specifies the minimum requirements for a Site-to-Site VPN connection of AES128, SHA1, and Diffie-Hellman group 2 in most AWS Regions, and AES128, SHA2, and Diffie-Hellman group 14 in the AWS GovCloud Regions. It also specifies pre-shared keys for authentication. You must modify the example configuration file to take advantage of additional security algorithms, Diffie-Hellman groups, private certificates, and IPv6 traffic. 

* Open the Amazon VPC console at https://console.aws.amazon.com/vpc/
* In the navigation pane, choose Site-to-Site VPN Connections.
* Select your VPN connection and choose Download Configuration.


### Creating a new IPsec VPN on pfsense

At `VPN > IPsec > Add`  Fill out the values from the text file that you just downloaded from AWS. It looks like this.

![vpn infra](/img/include/aws-vpn01.webp)

![vpn infra](/img/include/aws-vpn02.webp)

![vpn infra](/img/include/aws-vpn03.webp)

![vpn infra](/img/include/aws-vpn04.webp)

As with `Phase 1`, do the same for `Phase 2`. Read the values from the text file.

![vpn infra](/img/include/aws-vpn05.webp)

![vpn infra](/img/include/aws-vpn06.webp)

