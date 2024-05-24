# Setup environment

## Lab overview

In this lab, you will learn how to manage conditional in terraform configuration.

## Objectives

After you complete this lab, you will be able to:

-   Create multiple IP restrictions for an App Service
-   Understand how Terraform handles dynamic blocks

## Instructions

### Before you start

- Ensure Terraform (version >= 1.0.0) is installed and available from system's PATH.
- Ensure Azure CLI is installed.
- Check your access to the Azure Subscriptions and Resource Groups provided for this training.

### Exercise 1: Setup your environment

In your *main* Resource Group (the one tagged with attribute **layer** with value **main**), create a Storage Account, with a Blob container named **tfstate**

Clone the repository https://github.com/smartinez-cellenza/training-terraform-intermediate-labs-setup

```bash
git clone https://github.com/smartinez-cellenza/training-terraform-intermediate-labs-setup.git
cd training-terraform-intermediate-labs-setup
```

> This template contains a basic Terraform project configuration

In the *configuration/dev* folder, update the **backend.hcl** file :

- **resource_group_name**  = "the_name_of_your_main_resource_group"
- **storage_account_name** = "the_name_of_the_storage_account_you_just_created"
- **container_name**       = "tfstates"
- **key**                  = "dynamic.tfstate"

> This template use Partial backend configuration. The Storage Account just created is used for tfstate file persistence

In the *configuration/dev* folder, update the *dev.tfvars* file :

- **resource_group_name** = "the_name_of_your_main_resource_group"

### Exercise 2: Use conditional to change an argument based on condition

In the variable file, add the following variable

```go


variable "Kv" {
  type = object({
    KvName = string
    KvRgName = string
    KvLocation = string
    IsKvPublic = optional(bool,true) 
    KvEnableRbac = optional(bool,true) 
    KvNaClBypassConfig = optional(string,"AzureServices")
  })



```

In the main.tf file, add the following configuration

```go

resource "azurerm_key_vault" "Kv" {
  name = var.Kv.KvName
  resource_group_name = var.Kv.KvRgName
  location = var.Kv.KvLocation
  sku_name = "standard"
  tenant_id = data.azurerm_client_config.currentclientconfig.tenant_id
  enable_rbac_authorization = var.Kv.KvEnableRbac
  network_acls {
    bypass = var.Kv.KvNaClBypassConfig
    default_action = local.KvNaclDefaultAction
  }
  
}

```

Because we added a local, we need to add a new file named `local.tf` and specify the local:

```go


locals {
  KvNaclDefaultAction = var.Kv.IsKvPublic ? "Allow" : "Deny"
}


```

Also, because we added a reference to a data source, we need to add the folowing:

```go

data "azurerm_client_config" "currentclientconfig" {}

```

In the `dev.tfvars` file, add the value for the `Kv` variable:

```go

Kv = {
    KvLocation = "eastus"
    KvName = "<a_unique_key_vault_name>"
    KvRgName = "<your_resource_group_name>"
  }

```

Deploy the resources with the folowing command

```bash

az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve

```

At the end of the deployment, check the deployed key vault on Azure, in the network section.

![illustration1](/Img/001.png)

In the `dev.tfvars` file, add the following line to change the value of `Kv.IsKvPublic`:

```go

Kv = {
    KvLocation = "eastus"
    KvName = "<a_unique_key_vault_name>"
    KvRgName = "<your_resource_group_name>"
    IsKvPublic = false
  }


```

Run a plan to view the proposed change. a unique change should be displayed: 

```bash

Terraform will perform the following actions:

  # azurerm_key_vault.Kv will be updated in-place
  ~ resource "azurerm_key_vault" "Kv" {
        id                              = "/subscriptions/00000000-0000-0000-000000000000/resourceGroups/rsg-ciliumlabntw/providers/Microsoft.KeyVault/vaults/your_kv_name"
        name                            = "your_kv_name"
        tags                            = {}
        # (13 unchanged attributes hidden)

      ~ network_acls {
          ~ default_action             = "Allow" -> "Deny"
            # (3 unchanged attributes hidden)
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.

```

Apply the change and check again on the portal

![illustration2](/Img/002.png)


### Exercise 3: Use conditional to enable/disable a resource based on condition

A key vault can be used with rbac or with access policy. In the first casae, sepcific Azure rbac roles need to be assigned to the users so that they can access the elments in the key vault. In the second case, access policies have to be defined on the key vault to give access to users on the differents object types.

In the main file, add the following configuration:

```go

resource "azurerm_key_vault_access_policy" "KvAccessPolicy" {

  count = var.Kv.KvEnableRbac ? 0 : 1
  key_vault_id = azurerm_key_vault.Kv.id
  tenant_id    = data.azurerm_client_config.currentclientconfig.tenant_id
  object_id    = data.azurerm_client_config.currentclientconfig.object_id

  key_permissions = [
    "Get",
  ]

  secret_permissions = [
    "Get",
  ]
}

```

Run a plan and observe the change proposed.
In the `dev.tfvars`, change the `Kv.KvEnableRabc` to `false` and the `Kv.IsKvPublic` to `true`.
Re run an plan and an apply. Check the Key vault to the portal

![illustration3](/Img/003.png)

![illustration4](/Img/004.png)


Remove the resources using the command

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```