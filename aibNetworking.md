# Networking with Azure VM Image Builder

> **MAY 2020 SERVICE ALERT** - Existing users, please ensure you are compliant this [Service Alert by 26th May!!!](https://github.com/doug-mclelland/azvmimagebuilder#service-update-may-2020-action-needed-by-26th-may---please-review)

When the Azure Image Builder (AIB) runs, it starts Packer within the service, a build VM will be deployed into your subscription, into the AIB staging resource group, prefixed 'IT_', this is customized and turned into an image. 

You have two options when deploying Azure VM Builder:

1. Deploy without specifying an existing VNET
2. Deploy using an existing VNET (new), and no Public IP requirement.

In this article, we cover the following:
* Deploy without specifying an existing VNET
* Deploy using an existing VNET
    * What is Azure Private Link?
    * What is deployed during an image build?
    * Permissions AIB requires to use an existing VNET
    * Why deploy an additional VM deployment (proxy VM)?
    * Image Template parameters to support VNET
* Checklist for using your VNET with AIB

## Deploy without specifying an existing VNET
If you do not specify an existing VNET, then AIB will create a VNET and subnet in the staging resource group, in addition a public IP resource is used, with NSGs to restrict inbound traffic to the AIB service. The public IP is used to facilate the channel for the AIB cmds for the image build. Once the build had completed, the VM, public IP, disks, VNET are all deleted. To use this model, do not specify any VNET properties.

## Deploy using an existing VNET (new)
If you specify a VNET and subnet, then AIB will deploy the build VM to your chosen VNET, this means you can access resources that are accessible on your VNET. Of course, you can also create a silo'd VNET that is not connected to any other VNET too, and this will not use a Public IP either.

When the VM is deployed, a public IP is NOT deployed, communication from the AIB service to the build VM uses Azure Private Link technology instead.

We have end to end examples now you can try:
1. [Windows](https://github.com/doug-mclelland/azvmimagebuilder/tree/master/quickquickstarts/1a_Creating_a_Custom_Win_Image_on_Existing_VNET#create-a-windows-linux-image-allowing-access-to-an-existing-azure-vnet)
2. [Linux](https://github.com/doug-mclelland/azvmimagebuilder/tree/master/quickquickstarts/1a_Creating_a_Custom_Linux_Image_on_Existing_VNET#create-a-custom-linux-image-allowing-access-to-an-existing-azure-vnet)

### What is Azure Private Link?
Azure Private Link provides private connectivity from a virtual network to Azure platform as a service (PaaS), customer-owned, or Microsoft partner services. It simplifies the network architecture and secures the connection between endpoints in Azure by eliminating data exposure to the public internet. For more details, review the Private Link [documentation](https://docs.microsoft.com/en-us/azure/private-link/).

### Permissions AIB requires to use an existing VNET
The AIB SPN will require specific permissions to use an existing VNET, please review this [documentation](https://github.com/doug-mclelland/azvmimagebuilder/blob/master/aibPermissions.md#allowing-aib-to-customize-images-on-your-existing-vnets).

### What is deployed during an image build?
Using an existing VNET means AIB has to deploy an additional VM (proxy VM) and an Azure Load Balancer (ALB) that is connected to the Private Link. Traffic from the AIB service goes across the private link to the ALB, then the ALB communicates to the proxy VM using, port 60001 is for Linux OS's, and 60000 for Windows OS's. The proxy forwards commands to the build VM using port 22 for Linux OS's, or 5986 for Windows OS's.

> NOTE! The VNET must be in the same region as the AIB service region.

#### Why deploy an additional VM deployment (proxy VM)?
When a VM without public IP is behind an Internal Load Balancer, it doesn't have Internet access, and the ALB we use for VNET is internal, so we have to introduce the proxy VM to make sure build VM has Internet access for builds. However the NSG's associated, can be used to restrict the build VM access.

The deployed proxy VM size is Standard A1_v2 ,in addition to the build VM. The proxy VM is used to send commands between the AIB service and the build VM. The VM properties cannot be changed, such as size, or OS, which is Ubuntu 18.04, this is the same for both Windows and Linux image builds.

#### Image Template parameters to support VNET
```json
    "vnetConfig": {
        "subnetId": "/subscriptions/<subscriptionID>/resourceGroups/<vnetRgName>/providers/Microsoft.Network/virtualNetworks/<vnetName>/subnets/<subnetName>"
        }
    }
```
*vnetName* - Name of a pre-existing virtual network.
*subnetName* - Name of the subnet within the specified virtual network. Must be specified if and only if 'name' is specified.
*vnetRgName* - Name of the resource group containing the specified virtual network. Must be specified if and only if 'name' is specified.

Private Link service requires an IP from the given vnet and subnet. Currently, Azure does’t support Network Policies on these IPs. Hence, network policies need to be disabled on the subnet.  For more details, review the Private Link [documentation](https://docs.microsoft.com/en-us/azure/private-link/).

## Checklist for using your VNET with AIB
1. Allow Azure Load Balancer (ALB) to communicate with the proxy VM in an NSG
    * [AZ CLI Example](https://github.com/doug-mclelland/azvmimagebuilder/tree/master/quickquickstarts/1a_Creating_a_Custom_Linux_Image_on_Existing_VNET#add-nsg-rule-to-allow-the-aib-deployed-azure-load-balancer-to-communicate-with-the-proxy-vm)
    * [PowerShell Example](https://github.com/doug-mclelland/azvmimagebuilder/tree/master/quickquickstarts/1a_Creating_a_Custom_Win_Image_on_Existing_VNET#add-nsg-rule-to-allow-the-aib-deployed-azure-load-balancer-to-communicate-with-the-proxy-vm)
2. Disable Private Service Policy on subnet
    * [AZ CLI Example](https://github.com/doug-mclelland/azvmimagebuilder/tree/master/quickquickstarts/1a_Creating_a_Custom_Linux_Image_on_Existing_VNET#disable-private-service-policy-on-subnet)
    * [PowerShell Example](https://github.com/doug-mclelland/azvmimagebuilder/tree/master/quickquickstarts/1a_Creating_a_Custom_Win_Image_on_Existing_VNET#disable-private-service-policy-on-subnet)
3. Allow AIB to create an ALB and add VMs to the VNET
    * [AZ CLI Example](https://github.com/doug-mclelland/azvmimagebuilder/blob/master/aibPermissions.md#setting-aib-spn-permissions-to-allow-it-to-use-an-existing-vnet)
    * [PowerShell Example](https://github.com/doug-mclelland/azvmimagebuilder/blob/master/aibPermissions.md#setting-aib-spn-permissions-to-allow-it-to-use-an-existing-vnet-1)
4. Allow AIB to read/write source images and create images
    * [AZ CLI Example](https://github.com/doug-mclelland/azvmimagebuilder/blob/master/aibPermissions.md#setting-aib-spn-permissions-to-use-source-custom-image-and-distribute-a-custom-image)
    * [PowerShell Example](https://github.com/doug-mclelland/azvmimagebuilder/blob/master/aibPermissions.md#setting-aib-spn-permissions-to-use-source-custom-image-and-distribute-a-custom-image-1)
5. Ensure you are using VNET in the same region as the AIB service region.
