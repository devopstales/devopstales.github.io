---
title: Create Proxmox VM with Terraform
url: https://devopstales.github.io/cloud/proxmox-terraform/
date: 2023-04-20
keywords: terraform, Proxmox
---


In this post I will show you how how you can create a Proxmox VMs with Terraform.

<!--more-->

To ease the creation of the VMs in Proxmox I will use predefined VM templates. So our first task is create the template. To do this login to one of the proxmox nodes.

```bash
# installing libguestfs-tools only required once, prior to first run
sudo apt update -y
sudo apt install libguestfs-tools -y

# download a debian cloud-init disk image:
wget http://cloud.debian.org/images/cloud/bullseye/latest/debian-11-generic-amd64.qcow2

sudo virt-customize -a debian-11-generic-amd64.qcow2 --install qemu-guest-agent
sudo qm create 100200 --name "debian11.vm.shiwaforce.com" --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0

sudo qm importdisk 100200 debian-11-generic-amd64.qcow2 zfs-vm-none
sudo qm set 100200 --scsihw virtio-scsi-pci --scsi0 zfs-vm-none:vm-100200-disk-0
sudo qm set 100200 --boot c --bootdisk scsi0
sudo qm set 100200 --ide2 zfs-vm-none:cloudinit
sudo qm set 100200 --serial0 socket --vga serial0
sudo qm set 100200 --agent enabled=1
sudo qm template 100200

rm debian-11-generic-amd64.qcow2
```

## Determine Authentication Method (use API keys)

The next step is to create a service account in proxmox for authentication and generate an authentication token:

```bash
pveum role add TerraformProv -privs "Pool.Allocate VM.Console VM.Allocate VM.Clone VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Monitor VM.Audit VM.PowerMgmt Datastore.AllocateSpace Datastore.Audit"
pveum user add tfuser@pve
pveum aclmod / -user tfuser@pve -role TerraformProv
pveum user token add tfuser@pve terraform --privsep 0

# Please save the token secret as there isn't any way to fetch it at a later point.
```

## Create teh Terraform config files

```bash
mkdir terraform
cd terraform
```

Create backend config:

```json
nano backend.tf 
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "2.9.11"
    }
  }
  backend "local" {
  }
}
```

Create provider config:

```json
nano provider.tf
provider "proxmox" {
  pm_api_url = "https://proxmox.mydomain.intra:8006/api2/json"
  # api token id is in the form of: <username>@pam!<tokenId>
  pm_api_token_id = "tfuser@pve!terraform"
  # this is the full secret wrapped in quotes.
  pm_api_token_secret = var.PROXMOX_API_SECRET
  pm_tls_insecure     = true

  # debug log
  #  pm_log_enable = true
  #  pm_log_file   = "terraform-plugin-proxmox.log"
  #  pm_debug      = true
  #  pm_log_levels = {
  #    _default    = "debug"
  #    _capturelog = ""
  #  }
}
```

The auth token will be ised from the `PROXMOX_API_SECRET` linux environment variable. Now create the terraform variable file:

```json
nano vars.tf
variable "PROXMOX_API_SECRET" {
  type = string
}

variable "ssh_key" {
  default = "ssh-rsa AAAAB3NzaC1y..."
}

variable "proxmox_host" {
  default = "proxmox"
}
variable "template_name" {
  default = "debian11.vm.shiwaforce.com"
}

```

Create the main Terraform config file vit the resource definitions:

```json
touch main.tf 

nano k8s-master.tf 
resource "proxmox_vm_qemu" "k8s-master" {
  count       = 3
  name        = "k8s0${count.index + 1}.mydomain.intra"
  target_node = var.proxmox_host
  vmid        = "101${count.index + 1}"
  clone       = var.template_name
  os_type     = "cloud-init"
  cpu         = "kvm64"
  cores       = 2
  sockets     = 1
  memory      = 6144
  scsihw      = "virtio-scsi-pci"
  bootdisk    = "scsi0"

  disk {
    slot     = 0
    size     = "50G"
    type     = "scsi"
    storage  = "zfs-vm-etc"
    iothread = 1
  }

  network {
    model     = "virtio"
    bridge    = "vmbr0"
    tag       = 101
    firewall  = false
    link_down = false
  }

  ipconfig0 = "ip=192.168.10.1${count.index + 1}/22,gw=192.168.10.1"
  ciuser    = "terraform"
  sshkeys   = <<EOF
  ${var.ssh_key}
  EOF
}
```

If you created all the files it looks like this:

```bash
cd ..
tree .
.
└── terraform
    ├── backend.tf
    ├── provider.tf
    ├── outputs.tf
    ├── k8s-master.tf 
    └── vars.tf

1 directory, 5 files
```

Now we can create the VMs:

```bash
export PROXMOX_API_SECRET="9f881350-175c-4a09-a25c-bce726ada447"

# inicialise the terraform modules
terraform init --upgrade

# generate a terraform plan
terraform plan -out k8s-master.tfplan

# deploym the VMs
terraform apply -parallelism=1 -auto-approve k8s-master.tfplan

# cleanup
terraform plan -destroy -out k8s-master.destroy.tfplan
terraform apply k8s-master.destroy.tfplan
```

