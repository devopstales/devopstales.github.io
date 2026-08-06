---
title: Configure Kubernetes In-Tree vSphere Cloud Provider
url: https://devopstales.github.io/kubernetes/k8s-vmware/
date: 2020-10-14
keywords: kubernetes deployment, vmware, vSphere
---


In this post I will show you how can you use vmware for persistent storagi on K8S.

<!--more-->

{{< content "/filedir/kubernetes.html" >}}

### vSphere Configuration

* Create a folder for all the VMs in vCenter
 * In the navigator, select the data center
 * Right-click and select the menu option to create the folder.
 * Select: All vCenter Actions > New VM and Template Folder.
 * Move K8S vms to this folder
* The name of the virtual machine must match the name of the nodes for the K8S cluster.

![Example image](/img/include/k8s-vmware.webp)

### Set up the GOVC environment:

```bash
# on deployer
curl -LO https://github.com/vmware/govmomi/releases/download/v0.20.0/govc_linux_amd64.gz
gunzip govc_linux_amd64.gz
chmod +x govc_linux_amd64
cp govc_linux_amd64 /usr/bin/govc
echo "export GOVC_URL='vCenter IP OR FQDN'" >> /etc/profile
echo "export GOVC_USERNAME='vCenter User'" >> /etc/profile
echo "export GOVC_PASSWORD='vCenter Password'" >> /etc/profile
echo "export GOVC_INSECURE=1" >> /etc/profile
source /etc/profile
```

Add `disk.enableUUID=1` for all VM:

```bash
govc vm.info <vm>
govc ls /Datacenter/kubernetes/<vm-folder-name>
# example:
govc ls /Datacenter/kubernetes/k8s-01

govc vm.change -e="disk.enableUUID=1" -vm='VM Path'
# example:
govc vm.change -e="disk.enableUUID=1" -vm='/datacenter/kubernetes/k8s-01/k8s-m01'
```

VM Hardware should be at version 15 or higher. Upgrade if needed:

```bash
govc vm.option.info '/datacenter/kubernetes/k8s-01/k8s-m01' | grep HwVersion

govc vm.upgrade -version=15 -vm '/datacenter/kubernetes/k8s-01/k8s-m01'
```

### Create the required Roles

* Navigate in the vSphere Client - Menu > Administration > Roles
* Add a new Role and select the permissions required. Repeat for each role.

| Roles |	Privileges | Entities | Propagate to Children |
|:------:|:------:|:------:|:-------:|
| vcp-manage-k8s-node-vms | Resource.AssignVMToPoolVirtualMachine.Config.AddExistingDisk, VirtualMachine.Config.AddNewDisk, VirtualMachine.Config.AddRemoveDevice, VirtualMachine.Config.RemoveDisk, VirtualMachine.Config.SettingsVirtualMachine.Inventory.Create, VirtualMachine.Inventory.Delete | Cluster, Hosts, VM Folder | Yes |
|vcp-manage-k8s-volumes | Datastore.AllocateSpace, Datastore.FileManagement (Low level file operations) | Datastore | No |
| vcp-view-k8s-spbm-profile | StorageProfile.View (Profile-driven storage view) | vCenter | No |
| Read-only (pre-existing default role) | System.Anonymous, System.Read, System.View | Datacenter, Datastore Cluster, Datastore Storage Folder | No |

### Create a service account

* Create a vsphere user, or add a domain user, to provide access and assign the new roles to.

### Create vsphere.conf
Create the vSphere configuration file in /etc/kubernetes/vcp/vsphere.conf - you’ll need to create the folder.

```bash
nano /etc/kubernetes/vcp/vsphere.conf
[Global]
user = "k8s-user@vsphere.local"
password = "password for k8s-user"
port = "443"
insecure-flag = "1"

[VirtualCenter "10.0.1.200"]
datacenters = "DC-1"

[Workspace]
server = "10.0.1.200"
datacenter = "DC-1"
default-datastore = "vsanDatastore"
resourcepool-path = "ClusterNameHere/Resources"
folder = "kubernetes"

[Disk]
scsicontrollertype = pvscsi
```

### Modify the kubelet service
On master:

```bash
nano /etc/systemd/system/kubelet.service
[Service]
...
ExecStart=/usr/bin/docker run \
...
        /hyperkube kubelet \
...
--cloud-provider=vsphere --cloud-config=/etc/kubernetes/vsphere.conf    
```

On worker:

```bash
nano /etc/systemd/system/kubelet.service
[Service]
...
ExecStart=/usr/bin/docker run \
...
        /hyperkube kubelet \
...
--cloud-provider=vsphere  
```

### Modify container manifests
Add following flags to the kubelet service configuration (usually in the systemd config file), as well as the controller-manager and api-server container manifest files on the master node (usually in /etc/kubernetes/manifests).

```bash
nano /etc/kubernetes/manifests/kube-apiserver.yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
...
    - --cloud-provider=vsphere
    - --cloud-config=/etc/kubernetes/vsphere.conf
    volumeMounts:
    - mountPath: /etc/kubernetes/vcp
      name: vcp
      readOnly: true
...
  volumes:
  - hostPath:
      path: /etc/kubernetes/vcp
      type: DirectoryOrCreate
    name: vcp
```

Restart the services.

```bash
systemctl restart kubelet docker
```

### Add providerID

```bash
kubectl get nodes -o json | jq '.items[]|[.metadata.name, .spec.providerID, .status.nodeInfo.systemUUID]'

nano k8s-vmware-pacher.sh
DATACENTER='<Datacenter>'
FOLDER='<vm-folder-name>'
for vm in $(govc ls /$DATACENTER/vm/$FOLDER ); do
  MACHINE_INFO=$(govc vm.info -json -dc=$DATACENTER -vm.ipath="$vm" -e=true)
  # My VMs are created on vmware with upper case names, so I need to edit the names with awk
  VM_NAME=$(jq -r ' .VirtualMachines[] | .Name' <<< $MACHINE_INFO | awk '{print tolower($0)}')
  # UUIDs come in lowercase, upper case then
  VM_UUID=$( jq -r ' .VirtualMachines[] | .Config.Uuid' <<< $MACHINE_INFO | awk '{print toupper($0)}')
  echo "Patching $VM_NAME with UUID:$VM_UUID"
  # This is done using dry-run to avoid possible mistakes, remove when you are confident you got everything right.
  kubectl patch node $VM_NAME -p "{\"spec\":{\"providerID\":\"vsphere://$VM_UUID\"}}"
done

chmod +x openshift-vmware-pacher.sh
./openshift-vmware-pacher.sh
```

### Create vSphere storage-class

```bash
nano vmware-sc.yml
---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: "vsphere-standard"
provisioner: kubernetes.io/vsphere-volume
parameters:
    diskformat: zeroedthick
    datastore: "NFS"
reclaimPolicy: Delete

kubectl aplay -f vmware-sc.yml
```

