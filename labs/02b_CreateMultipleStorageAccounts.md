# Lab overview

In this lab, you will learn how to deploy multiple resources of same type using a loop in a single block.We will be deploying all resources into the same subscription (*main*).

- [Lab overview](#lab-overview)
  - [Objectives](#objectives)
  - [Instructions](#instructions)
    - [Before you start](#before-you-start)
    - [Exercise 1: Setup your environment (*tfstate* and project template)](#exercise-1-setup-your-environment-tfstate-and-project-template)
      - [Backend](#backend)
      - [Variables](#variables)
    - [Exercise 2: Deploy multiple storage accounts using a set](#exercise-2-deploy-multiple-storage-accounts-using-a-set)
      - [Variable set](#variable-set)
      - [Provider](#provider)
      - [Resources](#resources)
      - [Deploy](#deploy)
      - [Examine the tfstate](#examine-the-tfstate)
      - [`each.value` or `each.key`?](#eachvalue-or-eachkey)
      - [Remove resources](#remove-resources)
    - [Exercise 3: Deploy multiple storage accounts using a map](#exercise-3-deploy-multiple-storage-accounts-using-a-map)
      - [Resources](#resources-1)
      - [Deploy](#deploy-1)
      - [Examine the tfstate](#examine-the-tfstate-1)
      - [Remove resources](#remove-resources-1)

## Objectives

After you complete this lab, you will be able to:

- Create multiple Storage Accounts within a single block,
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
  key                 = "deploymultiplesa.tfstate"
```

> Define your backend using the Storage Account you created few minutes ago.

#### Variables

In the *configuration/dev* folder, update the `dev.tfvars` file:

```hcl
resource_group_name = "the_name_of_your_main_resource_group"
```

### Exercise 2: Deploy multiple storage accounts using a set

#### Variable set

In the `variables.tf` file, add a new variable:

```hcl
variable "storage_account_names" {
    type = set(string)
    description = "List of Storage Accounts to create"
}
```

> This variable is a set of string. It can only contain unique values.
> We will be using this variable to define the names of the (set of) Storage Accounts to create.

Define (a set of) values for the new variable in `dev.tfvars` under *configuration/dev* folder as:

```hcl
storage_account_names = [
  "a_unique_storage_account_name",
  "another_unique_storage_account_name",
  "a_third_unique_storage_account_name"
]
```

> Storage Accounts have a public FQDN based on their names. This requires the Storage Account names to be globally unique accross all Azure Resources.

#### Provider

**IMPORTANT**
We will be targeting here a deployment to what we used to call the *main* subscription in previous lab *01_DeployToMultipleSubscription*.
**Verify that the file `provider.tf` has the required provider block to address the creation of the resources.**

#### Resources

Reference the resource group to use (in *main* subscription) and add the Storage Account `resource` block in `main.tf` file:

```hcl
data "azurerm_resource_group" "self" {
  name = var.resource_group_name
}

resource "azurerm_storage_account" "example" {
  # we are looping on each value of storage_account_names
  for_each                 = var.storage_account_names
  name                     = each.key
  resource_group_name      = data.azurerm_resource_group.self.name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
}
```

> Note the `for_each` meta-argument in the storage account resource block:
>
> - it loops on the set of values we defined in `variables.tf`,
> - it uses `each.key` to get the name of every Storage Account.
>
> The `for_each` meta-argument accepts a set or a map, i.e. a collection value that has one element for each repetition.
>
> With `for_each`, Terraform creates an additional `each` object that can be used in expressions anf that two attributes:
>
> - `each.key`
> - `each.value`.
>
> In case a **set** is provided, `each.key` = `each.value`.
>
> To prevent unexpected behavior during conversion, the `for_each` argument does not implicitly convert lists or tuples (lists accepting elements of different types) to sets. Use Terraform expressions and functions to derive a suitable value (e.g. `flatten()` to get a list from a complex object and next `tomap()` to convert the list to a input map for `for_each`).

#### Deploy

Run the following commands to initialize your terraform environment and deploy the resources:

PowerShell

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
# use the '-reconfigure' option in case you already configured your local folder with a different backend (e.g. from previous labs)
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

Bash

```bash
az login
az account set --subscription "the_main_subscription_id"
export ARM_SUBSCRIPTION_ID="the_main_subscription_id"
# use the '-reconfigure' option in case you already configured your local folder with a different backend (e.g. from previous labs)
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal the created Storage Accounts.

#### Examine the tfstate

To examine resources present in the *tfstate* file, you can run the following command:

```powershell
terraform state list
```

Check the name of the resources which is suffixed with the storage account name.

> The `terraform state list` command can be used to list all the resources present in the *tfstate* file, and to display their internal (terraform) name.

#### `each.value` or `each.key`?

Update the name attribute of the `azurerm_storage_account` block to use `each.value` instead of `each.key`:

```hcl
resource "azurerm_storage_account" "example" {
  for_each                 = var.storage_account_names
  
  # we are now using each.value i/o each.key for the name
  name                     = each.value
  resource_group_name      = data.azurerm_resource_group.self.name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
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

### Exercise 3: Deploy multiple storage accounts using a map

#### Resources

In the `variables.tf` file, update the `storage_account_names` variable definition:

```hcl
variable "storage_account_names" {
  type = map(object({
    location                 = string
    account_replication_type = string
  }))
  description = "List of Storage Accounts to create"
}
```

> This variable is now a map of objects.
>
> Using a map allows you to specify multiple attributes for each item. In most real life cases, not only name needs to be defined for items such as in previous exercise, but also various attributes.

in the *configuration/dev* folder, update the `dev.tfvars` file content and replace the value of `storage_account_names` with:

```hcl
storage_account_names = {
    "a_unique_storage_account_name" = {
        location = "westeurope"
        account_replication_type = "GRS"
    }
    "another_unique_storage_account_name" = {
        location = "westeurope"
        account_replication_type = "LRS"
    }
    "a_third_unique_storage_account_name" = {
        location = "northeurope"
        account_replication_type = "GRS"
    }
}
```

> Storage Accounts have a public FQDN based on their names. This requires the Storage Account names to be globally unique accross all Azure Resources.

In `main.tf` file, replace the `resource` block with this one:

```hcl
resource "azurerm_storage_account" "example" {
  for_each = var.storage_account_names
  
  name                     = each.key
  resource_group_name      = data.azurerm_resource_group.self.name
  location                 = each.value.location
  account_tier             = "Standard"
  account_replication_type = each.value.account_replication_type
}
```

> Now the `for_each` meta-argument is looping over a map, so having different values for `each.key` and `each.value`:
>
> - `each.key` is used for the Storage Account name,
> - `each.value` (and its attributes) is used for location and replication type.
>
> Using a map adds more variabilisation for the input attributes!

#### Deploy

Run the following commands to deploy the resources:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal the created storage accounts.

#### Examine the tfstate

Run the `terraform state list` again and oberve the changes.

#### Remove resources

Remove the resources using the command:

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```
