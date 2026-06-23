# Lab overview

In this lab, you will learn how to deploy multiple resources of same type using a loop in a single block.  
We will also be using provider aliases to target different subscriptions for deploying resources.

- [Lab overview](#lab-overview)
  - [Objectives](#objectives)
  - [Instructions](#instructions)
    - [Before you start](#before-you-start)
    - [Exercise 1: Setup your environment (*tfstate* and project template)](#exercise-1-setup-your-environment-tfstate-and-project-template)
      - [Backend](#backend)
      - [Variables](#variables)
    - [Exercise 2: Deploy multiple virtual networks using a set](#exercise-2-deploy-multiple-virtual-networks-using-a-set)
      - [Variable set](#variable-set)
      - [Provider](#provider)
      - [Resources](#resources)
      - [Deploy](#deploy)
      - [Examine the tfstate](#examine-the-tfstate)
      - [`each.value` or `each.key`?](#eachvalue-or-eachkey)
      - [Remove resources](#remove-resources)
    - [Exercise 3: Deploy multiple virtual networks using a map](#exercise-3-deploy-multiple-virtual-networks-using-a-map)
      - [Resources](#resources-1)
      - [Deploy](#deploy-1)
      - [Examine the tfstate](#examine-the-tfstate-1)
      - [Remove resources](#remove-resources-1)
    - [Exercise 4: Deploy a hub \& spokes topology (ex. 3 follow-up)](#exercise-4-deploy-a-hub--spokes-topology-ex-3-follow-up)
      - [Resources](#resources-2)
      - [Deploy](#deploy-2)
      - [Remove resources](#remove-resources-2)

## Objectives

After you complete this lab, you will be able to:

- Create multiple Virtual Network within a single block,
- Understand how Terraform handles loops.

## Instructions

### Before you start

- Ensure Terraform (version ~> 1.13) is installed and available from system's PATH.
- Ensure Azure CLI is installed.
- Check your access to the Azure Subscriptions and Resource Groups provided for this training.

### Exercise 1: Setup your environment (*tfstate* and project template)

1/ Create the container for *tfstate*

In your *main* Resource Group (the one tagged with `layer` = `main`), create a Storage Account, with a Blob container named `tfstate` to store the *tfstate* file.

2/ Get project template

Clone the repository https://github.com/Anne-Gaelle-Cellenza/training-terraform-intermediate-labs-setup

```bash
git clone https://github.com/Anne-Gaelle-Cellenza/training-terraform-intermediate-labs-setup.git
cd training-terraform-intermediate-labs-setup
```

> This template contains a basic Terraform project configuration that:
>
> - uses a `data` resource group,
> - defines two variables `resource_group_name` and `location`,
> - contains a `configuration\dev` folder with *backend* and *tfvars* for dev.

3/ Configure the project template to use your environment

#### Backend

The project template uses a partial backend configuration: we don't define the backend configuration in the `terraform` block but in an external file, read at `terraform init` time.

In the *configuration/dev* folder, update the `backend.hcl` file as:

```hcl
  resource_group_name  = "the_name_of_your_main_resource_group"
  storage_account_name = "the_name_of_the_storage_account_you_just_created"
  container_name      = "tfstate"
  key                 = "deploymultiplevnet.tfstate"
```

> Define your backend using the Storage Account you created few minutes ago.

#### Variables

In the *configuration/dev* folder, update the `dev.tfvars` file:

```hcl
resource_group_name = "the_name_of_your_main_resource_group"
```

### Exercise 2: Deploy multiple virtual networks using a set

#### Variable set

In the `variables.tf` file, add a new variable:

```hcl
variable "vnet_to_create" {
  type        = set(string)
  description = "List of vnet to create"
}
```

> This variable is a set of string. It can only contain unique values.  
> We will be using this variable to define the names of the (set of) Virtual Networks to create.

Define (a set of) values for this new variable in `dev.tfvars` under *configuration/dev* folder as:

```hcl
vnet_to_create = [
  "vnet1",
  "vnet2",
  "vnet3"
]
```

#### Provider

**IMPORTANT**  
We will be targeting here a deployment to what we used to call the *feature* subscription in previous lab *01_DeployToMultipleSubscription*.  
**Verify that the file `provider.tf` has the required provider block to address the creation of the resources.**

#### Resources

Reference the resource group to use (in *feature* subscription) and add the Virtual Network `resource` block in `main.tf` file:

```hcl
data "azurerm_resource_group" "feature_rg" {
  name     = var.feature_rg
  provider = azurerm.feature
}

resource "azurerm_virtual_network" "feature_vnet" {
  # we are looping on each value of vnet_to_create
  for_each = var.vnet_to_create

  name                = each.key
  address_space       = ["10.10.0.0/24"]
  location            = var.location
  resource_group_name = data.azurerm_resource_group.feature_rg.name
  provider            = azurerm.feature
}
```

> Note the `for_each` meta-argument in the virtual netwrok resource block:
>
> - it loops on the set of values we defined in `variables.tf`,
> - it uses `each.key` to get the name of every VNet.
>
> The `for_each` meta-argument accepts a set or a map, i.e. a collection value that has one element for each repetition.  
> With `for_each`, Terraform creates an additional `each` object that can be used in expressions anf that two attributes:
>
> - `each.key`
> - `each.value`.
>
> In case a **set** is provided, `each.key` = `each.value`.
>
> To prevent unexpected behavior during conversion, the `for_each` argument does not implicitly convert lists or tuples to sets. Use Terraform expressions and functions to derive a suitable value (e.g. `flatten()` to get a list from a complex object and next `tomap()` to convert the list to a input map for `for_each`).

#### Deploy

Run the following commands to initialize your terraform environment and deploy the resources:

PowerShell

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
# use the '-reconfigure' option in case your local folder was previously configured with a different backend (e.g. from previous labs)
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

Bash

```bash
az login
az account set --subscription "the_main_subscription_id"
export ARM_SUBSCRIPTION_ID="the_main_subscription_id"
# use the '-reconfigure' option in case your local folder was previously configured with a different backend (e.g. from previous labs)
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal the created Virtual Networks.

#### Examine the tfstate

To examine resources present in the *tfstate* file, you can run the following command:

```powershell
terraform state list
```

Check the name of the resources which is suffixed with the VNet name.

> The `terraform state list` command can be used to list all the resources present in the *tfstate* file, and to and display their internal (terraform) name.

#### `each.value` or `each.key`?

Update the name attribute of the `azurerm_virtual_network` block to use `each.value` instead of `each.key`:

```hcl
resource "azurerm_virtual_network" "feature_vnet" {
  for_each = var.vnet_to_create

  # we are now using each.value i/o each.key for the name
  name                = each.value
  address_space       = ["10.10.0.0/24"]
  location            = var.location
  resource_group_name = data.azurerm_resource_group.feature_rg.name
  provider            = azurerm.feature
}
```

Run the following commands to update the configuration:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

Terraform does not identify any change. This is due to the type of input we gave to `for_each`: with a **set** input, `each.key` and `each.value` refer to the same value.

#### Remove resources

Remove the resources using the command:

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

### Exercise 3: Deploy multiple virtual networks using a map

#### Resources

In the `variables.tf` file, update the `vnet_to_create` variable definition:

```hcl
variable "vnet_to_create" {
  type = map(object({
    address_space = optional(string, "172.16.0.0/24")
    location = optional(string, "eastus")
  }))
  description = "List of virtual network to create"
}
```

> This variable is now a map of objects.
>
> A map is an unordered collection of key-value pairs where all values must be of the same type.  
> The object structure garantees the schema with named attributes ; it accepts multiple different attributes.  
> The map next allows you to define several objects with different attribute values and to iterate over them.

in the *configuration/dev* folder, update the `dev.tfvars` file content and replace the value of `vnet_to_create` with:

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

In `main.tf` file, replace the `resource` block with this one:

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

> Now the `for_each` meta-argument is looping over a map, so having different values for `each.key` and `each.value`:
>
> - `each.key` is used for the VNet name,
> - `each.value` (and its attributes) is used for address space and location.
>
> Using a map adds more variabilisation for the input attributes!

#### Deploy

Run the following commands to deploy the resources:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal the created virtual networks.

#### Examine the tfstate

Run the `terraform state list` again and oberve the changes.

#### Remove resources

Remove the resources using the command:

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

### Exercise 4: Deploy a hub & spokes topology (ex. 3 follow-up)

Let's now create a hub & spoke topology.
This requires an additional `hub` virtual network to which we will be peering the (spoke) VNets from previous exercise.

#### Resources

On top of the template files created in previous exercise, add the following in `main.tf` to create the hub network:

```hcl
resource "azurerm_virtual_network" "main_vnet" {
  name                = "hub"
  address_space       = ["10.0.0.0/24"]
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
}
```

> The `hub` resource doesn't call any provider alias so is using the default one (*main* subscription).

To create the peerings we need to define the 2-sides link for every spoke VNet to the `hub`.

```hcl
resource "azurerm_virtual_network_peering" "peer-main-to-feature" {
  for_each = var.vnet_to_create
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

#### Deploy

Run `terraform apply` with the proper arguments to create the resources.  

Check the created resources from Azure portal (spokes, hub and peerings).

#### Remove resources

Remove the resources using the command:

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```
