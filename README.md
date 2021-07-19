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
app="encrypt";                                echo $app
env="prod";                                   echo $env
l="eastus2";                                  echo $l
user_n_test="artiomlk";                       echo $user_n_test
user_pass_test="Password123!";                echo $user_pass_test

tags="env=$env app=$app";                     echo $tags
app_rg="rg-$app-$env";                        echo $app_rg

# ---
# NETWORK TOPOLOGY
# ---
vnet_pre1="172";                              echo $vnet_pre1
vnet_pre2="16";                               echo $vnet_pre2
vnet_pre="$vnet_pre1.$vnet_pre2";             echo $vnet_pre
vnet_n="vnet-$app-$env";                      echo $vnet_n
vnet_addr="$vnet_pre.0.0/16";                 echo $vnet_addr
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
