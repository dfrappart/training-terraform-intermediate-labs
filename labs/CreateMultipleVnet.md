# Setup environment

## Lab overview

In this lab, you will learn how to deploy multiple resources of the same type in a single block.

## Objectives

After you complete this lab, you will be able to:

-   Create multiple Virtual Network in a single block
-   Understand how Terraform handles loops

## Instructions

### Before you start

- Ensure Terraform (version >= 1.5.0) is installed and available from system's PATH.
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
- **key**                  = "deploymultiplevnet.tfstate"

> This template use Partial backend configuration. The Virtual Network just created is used for tfstate file persistence

In the *configuration/dev* folder, update the *dev.tfvars* file :

- **resource_group_name** = "the_name_of_your_main_resource_group"

### Exercise 2: Deploy multiple Virtual Network using a set

In the **variables.tf** file, add a new variable

```hcl
variable "vnet_to_create" {
  type        = set(string)
  description = "List of vnet to create"
}
```

> This variable is a set of string. It can only contains unique values.

> We will use this variable to handle the names of the Virtual Network to create to create

in the *configuration/dev* folder, add to the *dev.tfvars* file the variable value :

```hcl
vnet_to_create = [
  "vnet1",
  "vnet2",
  "vnet3"
]
```

In the *main.tf* file, add the following **resource** block to reference the Virtual Networks. 
**Verify that the file provider.tf has all the required provider block to address the creation of the resources.**

```hcl
resource "azurerm_virtual_network" "feature_vnet" {
  for_each = var.vnet_to_create

  name                = each.key
  address_space       = ["10.10.0.0/24"]
  location            = var.location
  resource_group_name = data.azurerm_resource_group.feature_rg.name
  provider            = azurerm.feature
}
```
Add a data source to reference the resource group.

```go

data "azurerm_resource_group" "feature_rg" {
  name     = var.feature_rg
  provider = azurerm.feature
}

```
> Notice the for_each attribute argument in the virtual netwrok resource block. Its value is the set we added in the variables

> The name attribute of the virtual network is each.key, referencing the occurence of the item in the set used in the for_each argument


Run the following commands in **src** folder to deploy the resources :

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
cd src
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal that the virtual network have been created

Run the following command

```powershell
terraform state list
```

> This command can be used to list all the resources present in the tfstate and display their internal name

The name of the resources is suffixed with the Storage Account name.

Update the attribute name of the `azurerm_virtual_network` block

```hcl
name                     = each.value
```

Run the following commands to update the configuration :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

Terraform does not identify any change. Since the value in the for_each is a set, **each.key** and **each.value** are the same

Remove the resources using the command

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

### Exercise 3: Deploy multiple virtual networks using a map

In the **variables.tf** file, update the **vnet_to_create** definition

```hcl
variable "vnet_to_create" {
  type = map(object({
    address_space = optional(string, "172.16.0.0/24")
    location = optional(string, "eastus")
  }))
  description = "List of virtual network to create"
}
```

> This variable is now a map of object

> Using a map allow to specify multiple attributes for each item. In most real life case, not only name needs to be defined, but also various attributes.


in the *configuration/dev* folder, update the *dev.tfvars* file content. Remove the previous value of **vnet_to_create** and add the following block

```hcl
vnet_to_create = {
    vnet1 = {
        location = "westeurope"
        address_space = "10.0.1.0/24"
    },
    vnet2 = {
        location = "eastus"
        address_space = "10.0.2.0/24"
    },
    vnet3 = {
        location = "westus"
        address_space = "10.0.3.0/24"
    }
}
```



In the *main.tf* file, replace the **resource** block by this one:

```hcl
resource "azurerm_virtual_network" "feature_vnet" {
  for_each = var.vnet_to_create

  name                = each.key
  address_space       = [each.value.address_space]
  location            = each.value.location
  resource_group_name = data.azurerm_resource_group.feature_rg.name
  provider            = azurerm.feature
}
```

> Notice the for_each attribute. Its value is the map we added in the variables

> The name attribute of the virtual network is each.key.

> location used the location attribute we defined in the map. This attribute can now be variabilised

> address_space used the address_space attribute we defined in the map. This attribute can now be variabilised


Run the following commands to deploy the resources :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal that the virtual network have been created

Run the `terraform state list` again and oberve the changes.

Remove the resources using the command

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```
### Exercise 4: Deploy a hub & spokes topology

In this exercice, we want to illustrte the creation of a hub & spoke topology.
This requires an additional virtual network that we will call `hub` and that all spokes network are peered to this hub network.

Keep the configuration written in the previous exercice. Add the following to create the hub network:

```go

resource "azurerm_virtual_network" "main_vnet" {
  name                = "hub"
  address_space       = ["10.0.0.0/24"]
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
}

```

To create the peerings, we need to define both side of it on the hub network, and the spokes:

```go

resource "azurerm_virtual_network_peering" "peer-main-to-feature" {
  for_each = var.vnets
  name                      = "peer-main-to-${azurerm_virtual_network.feature_vnet[each.key].name}"
  resource_group_name       = data.azurerm_resource_group.main.name
  virtual_network_name      = azurerm_virtual_network.main_vnet.name
  remote_virtual_network_id = azurerm_virtual_network.feature_vnet[each.key].id
}

resource "azurerm_virtual_network_peering" "peer-feature-to-main" {
  for_each = var.vnet_to_create
  name                      = format("peer-%s-to-main", azurerm_virtual_network.feature_vnet[each.key].name)
  resource_group_name       = data.azurerm_resource_group.feature_rg.name
  virtual_network_name      = azurerm_virtual_network.feature_vnet[each.key].name
  remote_virtual_network_id = azurerm_virtual_network.main_vnet.id
  provider                  = azurerm.feature
}

```

Run `terraform apply` with the proper arguments to create the resources.

Check on the portal that the spokes are all peered to the hub network.

Remove the resources using the command

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```