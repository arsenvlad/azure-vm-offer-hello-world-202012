# Create Azure VM Image with Packer - Shared Image Gallery (SIG) Destination

## Prepare

Download Packer binary <https://www.packer.io/downloads>

Review Packer templates

* [packer-sig-ubuntu.json](packer-sig-ubuntu.json)

Set environment variables with credentials of Service Principal that has Contributor role on the Azure subscription

```bash
set AZURE_SUBSCRIPTION_ID=
set AZURE_TENANT_ID=
set AZURE_CLIENT_ID=
set AZURE_CLIENT_SECRET=
```

![Packer - SIG Destination](../images/packer-sig.png)

Create Shared Image Gallery or skip if want to use an existing one

```bash
# Create resource group
az group create --name avsig200 --location eastus2

# Create shared image gallery resource
az sig create --resource-group avsig200 --gallery-name avgallery200

# Create shared image gallery image definition
az sig image-definition create --resource-group avsig200 --gallery-name avgallery200 --gallery-image-definition avimage200 --os-type Linux --publisher avpublisher200  --offer avoffer200 --sku avsku200
```

**Execute Packer build** to create Shared Image Gallery image version

```bash
packer build -var location=eastus2 -var sig_gallery_resource_group=avsig200 -var sig_gallery_name=avgallery200 -var sig_image_name=avimage200 -var sig_image_version=1.0.0 packer-sig-ubuntu.json
```

Create test VM from the Shared Image Gallery image

```bash
# Create resource group for the test VM
az group create --name avtestvm200 --location eastus2

# Create test VM with public IP
az vm create --resource-group avtestvm200 --name ubuntuvm --image "/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avsig200/providers/Microsoft.Compute/galleries/avgallery200/images/avimage200/versions/1.0.0" --admin-username azureuser --admin-password "Password@123"
```

Run certification tool against the VM

* [Certification Test Tool for Azure Certified](https://www.microsoft.com/download/details.aspx?id=44299)
* [Using Self-Test API to validate VM images for publishing in Azure Marketplace](https://arsenvlad.medium.com/using-self-test-api-to-validate-vm-images-for-publishing-in-azure-marketplace-e7ac2e0b4d6e)

Generate VHD SAS URL from the Shared Gallery Image

> For more details and screenshots, see this [blog post](https://arsenvlad.medium.com/creating-vhd-azure-blob-sas-url-from-azure-managed-image-2be0e7c287f4) which shows how to use the same approach for creating VHD SAS URL from a managed image by converting it to Shared Image Gallery first.

```bash
# Create Azure Managed Disk from Shared Image Gallery
az disk create --resource-group avsig200 --location eastus2 --name my-disk-from-image --gallery-image-reference /subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avsig200/providers/Microsoft.Compute/galleries/avgallery200/images/avimage200/versions/1.0.0

# Generate SAS URL for the Managed Disk
az disk grant-access --resource-group avsig200 --name my-disk-from-image --duration-in-seconds 36000 --access-level Read

# Copy Azure Managed Disk to Azure Blob Storage Container

# Create storage account
az storage account create --resource-group avsig200 --name avvhdstorage200 --location eastus2 --sku Standard_LRS --kind StorageV2 --access-tier Hot

# Create storage container
az storage container create --resource-group avsig200 --account-name avvhdstorage200 --name myvhds

# Create writable SAS URL for the container
az storage container generate-sas --account-name avvhdstorage200 --name myvhds --permissions acw --expiry "2020-12-01T00:00:00Z"

# Copy managed disk using its SAS URL to the container
azcopy copy "https://md-dcbqwqxfvbjb.blob.core.windows.net/f3gcv45nzg0z/abcd?sv=2018-03-28&sr=b&si=01897e7a-826c-4cd7-8229-0b0e1e30e5ec&sig=MhjTJ60QbJpegOhxLDHynHGkh935D6e7zpOyqIHd1gE%3D" "https://avvhdstorage200.blob.core.windows.net/myvhds/myimage1.vhd?se=2020-12-01T00%3A00%3A00Z&sp=acw&sv=2018-11-09&sr=c&sig=xIQxuK%2BfjUgAXSsXF%2B6fzY05GbNfUWuT3owl8LhhHTc%3D"

# Create SAS for the VHD to use in Partner Center
az storage container generate-sas --account-name avvhdstorage200 --name myvhds --permissions rl --start "2020-12-01T00:00:00Z" --expiry "2020-12-31T00:00:00Z"

# Append SAS token to the URL of the container to use in Azure Marketplace
# For example: https://avvhdstorage200.blob.core.windows.net/myvhds/myimage1.vhd?SAS_SIGNATURE_HERE
```
