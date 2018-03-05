# Azure image automation demo: container-like automation for standard VM images

Working with containers gives you new experiences in area of automation. When you need to go back to world of VMs (eg. because your applications are not ready for such move yet), you might miss quite a few things:
* You can easily pass information to containers with environmental variables or Kubernetes ConfigMaps
* Container images are immutable and come with all dependencies preinstalled
* Image creation process is fully automated (for example with Dockerfile)
* Known good image is starting point for your image (FROM statement in Dockerfile)
* Kubernetes Secret is easy and secure way to pass sensitive information to containers

There are way more advantages of containers beyond this, but things mentioned in list are not unique to containers. With right tooling you can use similar principles with VMs:
* You can pass information including simple data, configuration files, scripts or binaries to VM during provisioning time using cloud-init or Azure VM Extensions
* You can build immutable (preinstalled) images automatically, consistently and using starting image with Packer
* There is easy and very secure way to access sensitive information from your VM with Azure Managed Service Idenitity and Azure Key Vault

In this demo we are going to explore those options.

- [Azure image automation demo: container-like processes with standard VMs](#azure-image-automation-demo-container-like-processes-with-standard-vms)
    - [Standardized cloud-init for Linux in any cloud](#standardized-cloud-init-for-linux-in-any-cloud)
    - [Azure native solution with proprietary agent](#azure-native-solution-with-proprietary-agent)
    - [Automated image creation with Packer](#automated-image-creation-with-packer)
    - [Passing secrets with Azure Key Vault](#passing-secrets-with-azure-key-vault)


## Standardized cloud-init for Linux in any cloud
Standardized open source way to bootstrap VMs is cloud-init project that is working well with multiple clouds. It comes with many different modules to manage SSH keys, users, groups, install packages, create files or run scripts and much more. It is currently (as of March 2018) supported in Azure stock images for Ubuntu, CoreOS and in preview for certain CentOS and RHEL images (https://docs.microsoft.com/en-us/azure/virtual-machines/linux/using-cloud-init).

Key advantage is support with multiple clouds. On the other hand there is no Windows support (natively) and solution is not as tightly integrated to Azure portal.

Look at cloud-init.yaml. In our demo we are doing following provisioning steps:
* update repo
* install Apache2
* create new user group
* create new user and assign to group (note we are not specifying password - you can do this in form of hash, but it is not considered very secure, so for real situations use SSH keys)
* create new page for our web server

We are going to create resource groups, Ubuntu VM and pass cloud-init configuration via --custom-data.

```
az group create -n automation -l westeurope
az vm create -n testvm \
    -g automation \
    --image ubuntults \
    --public-ip-address-dns-name mujimagetest \
    --nsg "" \
    --admin-username tomas \
    --admin-password Azure12345678 \
    --authentication-type password \
    --size Standard_A1_v2 \
    --custom-data ./cloud-init.yaml
```
As soon as creation is complete log to your VM and tailf syslog to see cloud-init operations.

```
ssh tomas@mujimagetest.westeurope.cloudapp.azure.com 
tailf /var/log/syslog | grep cloud-init
```

Let's make sure provisioning was done properly.

```
tomas@testvm:~$ curl 127.0.0.1
<h1>This is my super page!</h1>

tomas@testvm:~$ grep newuser /etc/passwd
newuser:x:1001:1003:New User:/home/newuser:

tomas@testvm:~$ grep my-users /etc/group
my-users:x:1001:newuser
```

We can now delete this resource group.
```
az group delete -y -n automation
```

## Azure native solution with proprietary agent
Azure comes with Azure-specific agent and concept of VM Extensions which allow this agent to get enhanced with additional provisioning modules. This includes CustomScriptExtension that is available for both Linux and Windows images. There are couple of advantages compared to cloud-init:

* very good support for both Linux and Windows
* deeply integrated with Azure (status visible in portal)
* does not need to run only during provisioning, can be added at any point in VM lifecycle
* built-in mechanism to copy files of any size to VM
* built-in secure access to Azure blob storage

First we are going to create resource group and Windows VM.

```
az group create -n automation -l westeurope
az vm create -n testwinvm \
    -g automation \
    --image Win2016Datacenter \
    --public-ip-address-dns-name mujwinimagetest \
    --nsg "" \
    --admin-username tomas \
    --admin-password Azure12345678 \
    --size Standard_A2_v2 \
```

Custom script extension needs to copy files (scipts, configurations, binaries) from some URLs. For private use it is best to create Azure Storage account, copy files there and lever secure built-in mechanism to pass credentials.

Let's create storage account, get key, create container and upload two files.
```
az storage account create -n tomasautostore \
    -g automation \
    --sku Standard_LRS

export storagekey=$(\
    az storage account keys list -n tomasautostore \
        -g automation \
        --query [0].value \
        -o tsv \
    )

az storage container create -n deploy \
    --account-name tomasautostore \
    --account-key $storagekey

az storage blob upload -f ./index.html \
    -c deploy \
    -n index.html \
    --account-name tomasautostore \
    --account-key $storagekey

az storage blob upload -f ./install.ps1 \
    -c deploy \
    -n install.ps1 \
    --account-name tomasautostore \
    --account-key $storagekey

```

We have uploaded web page and our installation PowerShell script (add IIS, copy new home page). Once VM is up and running we will create Extension, pass information about files to download, set command to execute (that will be our script) and pass storage credentials.

```
az vm extension set -n CustomScriptExtension  \
    --publisher Microsoft.Compute \
    --version 1.8 \
    --vm-name testwinvm \
    --resource-group automation \
    --settings '{"fileUris":["https://tomasautostore.blob.core.windows.net/deploy/install.ps1","https://tomasautostore.blob.core.windows.net/deploy/index.html"]}' \
    --protected-settings '{"commandToExecute":"powershell.exe -File install.ps1","storageAccountName":"tomasautostore","storageAccountKey":"'$storagekey'"}' --debug
```

Once extension finishes checkout results.

```
curl mujwinimagetest.westeurope.cloudapp.azure.com
```

We can now delete resource group.

```
az group delete -y -n automation
```

## Automated image creation with Packer
While installation of software and dependencies during provision time is certainly possible with tools like cloud-init, VM Extensions, Ansible, Chef or Puppet, it comes with key disadvantage - it can take quite a lot of time to complete. For example if you would like to leverage auto-scaling based on actual user load and it takes 20 minutes to privision new VM you are not able to react quickly enough and also you need to pay for 20 minutes of VM time without bringing any value to your users during that time. 

In world of containers solution is to prepare images strictly **before** deployment by using Dockerfile to automate creation of container image. From that point container image is considered immutable, much like preinstalled appliance. You can leverage the same strategy with Packer. This tool helps you to automate and standardize creation of images in any cloud so long process of provisioning software is taken during image preparation, but immutable image is used later on. This gives you more efficiency and quicker startup times.

Packer automates the following workflow:
* Base image is used to create temporary VM (for example in Azure)
* Packer connects to VM and start provisioning phase with scripting, Ansible or other options
* After provisioning Packer does sanitization, turn of VM and capture it as new image
* Packer delete all temporary resources

Also make sure you are not using Packer to set configurations that you want to modify for different environments or embedd secrets into image - combine Packer with other techniques mentioned here to deliver things like feature flags or connection strings.

First let's download Packer binary.
```
wget https://releases.hashicorp.com/packer/1.2.1/packer_1.2.1_linux_amd64.zip
sudo unzip packer_1.2.1_linux_amd64.zip -d /usr/bin
rm packer_1.2.1_linux_amd64.zip
packer --verison
```

Check packerImage.json and fill in your service principal details (just for demo! Make sure you store your credentials outside of main automation script so you can save it securely to version control). Builders section is about automating Azure (or other environments at the same time) while provisioners section uses scripting to install apache2 and create home page. Output of Packer will be custom image.
```
az group create -n autoimage -l westeurope
packer build packerImage.json
```

Look into logs to see what Packer is doing. Temporary resource group and VM is created, then Packer logs in, run the script and finaly create image and destroy temporary resources.
```
==> azure-arm: Running builder ...
    azure-arm: Creating Azure Resource Manager (ARM) client ...
==> azure-arm: Creating resource group ...
==> azure-arm:  -> ResourceGroupName : 'packer-Resource-Group-h29nchvr0y'
...
==> azure-arm: Getting the VM's IP address ...
...
==> azure-arm: Waiting for SSH to become available...
...
==> azure-arm: Provisioning with shell script: /tmp/packer-shell350676811
...
==> azure-arm: Powering off machine ...
...
==> azure-arm: Capturing image ...
...
==> azure-arm: Deleting resource group ...
...
==> Builds finished. The artifacts of successful builds are:
--> azure-arm: Azure.ResourceManagement.VMImage:

ManagedImageResourceGroupName: autoimage
ManagedImageName: myPackerImage
ManagedImageLocation: westeurope
```

Let's make sure we see this new custom image.
```
$ az image list -g autoimage -o table
Location    Name           ProvisioningState    ResourceGroup
----------  -------------  -------------------  ---------------
westeurope  myPackerImage  Succeeded            autoimage
```

We now have preinstalled image that can be easily used with Azure Virtual machine Scale Set. Let's create load-balanced pool of 3 VMs, configure balancing rules and test it.
```
az group create -n scaleset -l westeurope
az vmss create -n myscaleset \
    -g scaleset \
    --image $(az image show -n myPackerImage -g autoimage --query id -o tsv) \
    --vm-sku Standard_A1_v2 \
    --instance-count 3 \
    --admin-username tomas \
    --admin-password Azure1234567 \
    --authentication-type password \
    --public-ip-address-dns-name tomaswebscale \
    --lb tomaslb \
    --backend-pool-name mypool \
    --public-ip-address lbip

az network lb rule create \
    --resource-group scaleset \
    --name myLoadBalancerRuleWeb \
    --lb-name tomaslb \
    --backend-pool-name mypool \
    --backend-port 80 \
    --frontend-ip-name loadBalancerFrontEnd \
    --frontend-port 80 \
    --protocol tcp

curl tomaswebscale.westeurope.cloudapp.azure.com
```

We can now delete all resource groups
```
az group delete -n scaleset -y --no-wait
az group delete -n autoimage -y --no-wait
```

## Passing secrets with Azure Key Vault
Most of previously used methods are not the most secure ways to pass secrets such as connection strings, passwords or certificates to our VMs. Not only transfer itself is not secure enough, but keeping secrets as part of automation scripts is also pretty bad practice. We want to separate lifecycle of secrets so we can put all automation files into version control system with no risk of leaking sensitive information.

In this demo we will use Azure Key Vault to securely store secrets outside of automation process. In order to access secrets by VM we will use Managed Service Identity, which is automated creation of Azure Active Directory service account for VM and secure delivery of authentication tokens within VM. Using this mechanism we will get access to Key Vault and read our secrets.

First create Ubuntu VM with managed identity.
```
az group create -n automation -l westeurope
az vm create -n testvm \
    -g automation \
    --image ubuntults \
    --public-ip-address-dns-name tomassecretstest \
    --nsg "" \
    --admin-username tomas \
    --admin-password Azure12345678 \
    --authentication-type password \
    --size Standard_A1_v2 \
    --assign-identity [system] \
    --no-wait
```

Now create Azure Key Vault and store our secret.
```
az keyvault create -n tomaskeyvault \
    -g automation

az keyvault secret set -n mysecret \
     --vault-name tomaskeyvault \
     --value thisIsMySecretString
```

Get VM principal ID (VM service account in AAD) and allow access to Key Vault.
```
export principalid=$(az vm show -n testvm \
    -g automation \
    --query identity.principalId \
    -o tsv)

az keyvault set-policy -n tomaskeyvault \
    -g automation \
    --object-id $principalid \
    --secret-permissions get
```

SSH to VM, authenticate to get Key Vault token and read secret. API calls are returning JSON, so we will use jq to parse it in CLI.
```
ssh tomas@tomassecretstest.westeurope.cloudapp.azure.com 

sudo apt install jq
export token=$(curl -s http://localhost:50342/oauth2/token \
    --data "resource=https://vault.azure.net" \
    -H Metadata:true \
    | jq -r .access_token)

curl -s https://tomaskeyvault.vault.azure.net/secrets/mysecret?api-version=2016-10-01 \
    -H "Authorization: Bearer $token" \
    | jq -r .value
```

We can delete resource group now.
```
az group delete -n automation -y --no-wait
```