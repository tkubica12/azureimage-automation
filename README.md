# Azure image automation demo

## Standardized cloud-init for Linux in any cloud
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

ssh tomas@mujimagetest.westeurope.cloudapp.azure.com 
tailf /var/log/syslog | grep cloud-init

az group delete -y -n automation


tomas@testvm:~$ curl 127.0.0.1
<h1>This is my super page!</h1>
tomas@testvm:~$ grep newuser /etc/passwd
newuser:x:1001:1003:New User:/home/newuser:
tomas@testvm:~$ grep my-users /etc/group
my-users:x:1001:newuser

## Azure native solution with proprietary agent

az group create -n automation -l westeurope
az vm create -n testwinvm \
    -g automation \
    --image Win2016Datacenter \
    --public-ip-address-dns-name mujwinimagetest \
    --nsg "" \
    --admin-username tomas \
    --admin-password Azure12345678 \
    --size Standard_A2_v2 \

az vm extension set -n CustomScriptExtension  \
    --publisher --publisher Microsoft.Compute \
    --version 1.8 \
    --vm-name testwinvm \
    --resource-group automation \
    --protected-settings '{"username":"user1", "ssh_key":"ssh_rsa ..."}'
    --settings '{"fileUris": ["https://pmcstorage100.blob.core.windows.net/scripts01/script01.sh"], "commandToExecute":"./install.ps1"}' 


curl mujwinimagetest.westeurope.cloudapp.azure.com

az group delete -y -n automation

## Automated image creation with Packer