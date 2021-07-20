# azEncryption

## Useful Commands

| Command                                                              | Description                                     |
| -------------------------------------------------------------------- | ----------------------------------------------- |
| `az login`                                                           | login to azure with your account                |
| `az account list --output table`                                     | display available subscription                  |
| `az account set --subscription xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | use subscriptionID                              |
| `az account show --output table`                                     | check the subscriptions currently in use        |
| `az group list -o table`                                             | -                                               |
| `az account list-locations -o table`                                 | -                                               |
| `az aks get-versions --location eastus2 -o table`                    | -                                               |
| `export MSYS_NO_PATHCONV=1`                                          | avoids the C:/Program Files/Git/ being appended |

## Instructions

### Connect to our azure subscription

```bash
# Login to azure
az login
# Display available subscriptions
az account list --output table
# Switch to the subscription where we want to work on
az account set --subscription xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# Check whether we are on the right subscription
az account show --output table
# If using Git Bash avoid C:/Program Files/Git/ being appended to some resources IDs
export MSYS_NO_PATHCONV=1
```

---

### Setup reusable variables

```bash
# ---
# Main Vars
# ---
app="encrypt";                                              echo $app
env="prod";                                                 echo $env
l="eastus2";                                                echo $l
user_n_test="artiomlk";                                     echo $user_n_test
user_pass_test="Password123!";                              echo $user_pass_test

tags="env=$env app=$app";                                   echo $tags
app_rg="rg-$app-$env";                                      echo $app_rg

# ---
# NETWORK TOPOLOGY
# ---
vnet_pre1="173";                                            echo $vnet_pre1
vnet_pre2="16";                                             echo $vnet_pre2
vnet_pre="$vnet_pre1.$vnet_pre2";                           echo $vnet_pre
vnet_n="vnet-$app-$env";                                    echo $vnet_n
vnet_addr="$vnet_pre.0.0/16";                               echo $vnet_addr

# ---
# Key Vault
# ---
kv_rg="rg-$app-kv-$env";                                    echo $kv_rg
kv_n="kv-$app-$env";                                        echo $kv_n

# ---
# VMs
# ---
#Topology
snet_vm_n="snet-$app-vms-$env";                             echo $snet_vm_n
snet_vm_addr="$vnet_pre.2.0/24";                            echo $snet_vm_addr
nsg_vm_n="nsg-$app-vms-$env";                               echo $nsg_vm_n
#Linux VMs
vm_linux_n="vm-$app-linux-$env";                            echo $vm_linux_n
vm_linux_img="UbuntuLTS ";                                  echo $vm_linux_img
vm_linux_size="Standard_D2S_V3";                            echo $vm_linux_size
#Windows VMs
vm_windows_n="vm-$app-$env";                                echo $vm_windows_n
vm_windows_img="win2016datacenter";                         echo $vm_windows_img
vm_windows_size="Standard_D2S_V3";                          echo $vm_windows_size
```

---

### Create Main Resource Group

```bash
# Create a resource group where our app resources will be created, e.g. AKS, ACR, vNets...
az group create \
--name $app_rg \
--location $l \
--tags $tags
```

---

### Create Main vNet

```bash
# Main vNet
az network vnet create \
--name $vnet_n \
--resource-group $app_rg \
--address-prefixes $vnet_addr \
--location $l \
--tags $tags
```

---

### Create Key Vault with Azure Disk Encryption enabled

```bash
# Key Vault Resource Group
az group create \
--name $kv_rg \
--location $l \
--tags $tags

# Create AzDiskEncryption KeyVault
az keyvault create \
--name $kv_n \
--resource-group $kv_rg \
--location $l \
--enabled-for-disk-encryption \
--tags $tags
```

---

### Encrypt Existing Linux & Windows VMs

```bash
# ---
# sNet Topology
# ---

# VMs NSG with Default rules
az network nsg create \
--resource-group $app_rg \
--name $nsg_vm_n \
--location $l \
--tags $tags

# VMs Subnet
az network vnet subnet create \
--resource-group $app_rg \
--vnet-name $vnet_n \
--name $snet_vm_n \
--address-prefixes $snet_vm_addr \
--network-security-group $nsg_vm_n

# ---
# Linux
# ---

# Create a Linux VM without Encryption
az vm create \
--resource-group $app_rg \
--name $vm_linux_n \
--vnet-name $vnet_n \
--subnet $snet_vm_n \
--image $vm_linux_img \
--size $vm_linux_size \
--admin-username $user_n_test \
--generate-ssh-keys \
--public-ip-address "" \
--nsg "" \
--nsg-rule NONE \
--tags $tags

# Encrypt Linux VM
KV_ID=$(az keyvault show --resource-group $kv_rg --name $kv_n  --query id --out tsv); echo $KV_ID
az vm encryption enable \
--name $vm_linux_n \
--resource-group $app_rg \
--disk-encryption-keyvault $KV_ID \
--volume-type ALL

# Check Encryption
az vm encryption show \
--name $vm_linux_n \
--resource-group $app_rg

# ---
# Windows
# ---

# Create a Windows VM without Encryption
az vm create \
--resource-group $app_rg \
--name $vm_windows_n \
--vnet-name $vnet_n \
--subnet $snet_vm_n \
--image $vm_windows_img \
--size $vm_windows_size \
--admin-username $user_n_test \
--admin-password $user_pass_test \
--public-ip-address "" \
--nsg "" \
--nsg-rule NONE \
--tags $tags

# Encrypt Windows VM
KV_ID=$(az keyvault show --resource-group $kv_rg --name $kv_n  --query id --out tsv); echo $KV_ID
az vm encryption enable \
--name $vm_windows_n \
--resource-group $app_rg \
--disk-encryption-keyvault $KV_ID \
--volume-type ALL

# Check Windows Encryption
az vm encryption show \
--name $vm_windows_n \
--resource-group $app_rg
```

---

### Encrypt Existing Linux & Windows VM Scale Sets

```bash
# ---
# Windows VMSS
# ---
# Windows VM Scale Sets
az vmss create \
--name $vm_windows_n \
--resource-group $app_rg \
--image $vm_windows_img \
--vm-sku $vm_windows_size \
--storage-sku StandardSSD_LRS \
--vnet-name $vnet_n \
--subnet $snet_vm_n \
--instance-count 3 \
--upgrade-policy-mode Automatic \
--single-placement-group false \
--platform-fault-domain-count 1 \
--admin-username $user_n_test \
--admin-password $user_pass_test \
--public-ip-address "" \
--nsg "" \
--tags $tags

# Encrypt Windows VMSS
KV_ID=$(az keyvault show --resource-group $kv_rg --name $kv_n  --query id --out tsv); echo $KV_ID
az vmss encryption enable \
-g $app_rg  \
-n $vm_windows_n \
--disk-encryption-keyvault $KV_ID \
--volume-type ALL

# Check Windows VMSS Encryption
az vmss encryption show \
--name $vm_windows_n \
--resource-group $app_rg

# ---
# Linux VMSS
# ---
# Linux VM Scale Sets
az vmss create \
--resource-group $app_rg  \
--name $vm_linux_n \
--image $vm_linux_img \
--vnet-name $vnet_n \
--subnet $snet_vm_n \
--instance-count 3 \
--upgrade-policy-mode automatic \
--admin-username $user_n_test \
--generate-ssh-keys \
--public-ip-address "" \
--nsg "" \
--data-disk-sizes-gb 32 \
--tags $tags

# Prepare the data disk for use with the Custom Script Extension
az vmss extension set \
--publisher Microsoft.Azure.Extensions \
--version 2.0 \
--name CustomScript \
--resource-group $app_rg \
--vmss-name $vm_linux_n \
--settings '{"fileUris":["https://raw.githubusercontent.com/Azure-Samples/compute-automation-configurations/master/prepare_vm_disks.sh"],"commandToExecute":"./prepare_vm_disks.sh"}'

# Encrypt Linux VMSS
KV_ID=$(az keyvault show --resource-group $kv_rg --name $kv_n  --query id --out tsv); echo $KV_ID
az vmss encryption enable \
-g $app_rg  \
-n $vm_linux_n \
--disk-encryption-keyvault $KV_ID \
--volume-type DATA

# Check Linux VMSS Encryption
az vmss encryption show \
--name $vm_linux_n \
--resource-group $app_rg
```

---

---

### Clean up resources

```bash
az keyvault delete --name $kv_n --resource-group $app_rg
az keyvault purge --name $kv_n --location $l
az group delete -n $kv_rg -y --no-wait
az group delete -n $app_rg -y --no-wait
```
