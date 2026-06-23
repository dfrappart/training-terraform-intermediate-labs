# Lab overview

In this lab, you will learn how to deploy resources to multiple subscriptions in the same template.After configuration of the backend (in *main* subscription), we will be deploying a VNet into two different subscriptions, *main* and *feature* and a peering between the VNets.

- [Lab overview](#lab-overview)
  - [Objectives](#objectives)
  - [Instructions](#instructions)
    - [Before you start](#before-you-start)
    - [Exercise 1: Setup your environment (*tfstate* and project template)](#exercise-1-setup-your-environment-tfstate-and-project-template)
      - [Backend](#backend)
      - [Variables](#variables)
    - [Exercise 2: Configure a new provider for the second targetted subscription (tagged as *feature*)](#exercise-2-configure-a-new-provider-for-the-second-targetted-subscription-tagged-as-feature)
    - [Exercise 3: Create the Virtual Networks](#exercise-3-create-the-virtual-networks)
    - [Exercise 4: Create peering](#exercise-4-create-peering)
    - [Exercise 5: Remove resources](#exercise-5-remove-resources)

## Objectives

After you complete this lab, you will be able to:

- Deploy two Virtual Networks over two subscriptions and create a peering between them,
- Understand how Terraform handles multiple subscriptions using provider and alias.

## Instructions

### Before you start

- Ensure Terraform (version ~> 1.13.0) is installed and available from system PATH.
- Ensure Azure CLI is installed.
- Check your access to the **two** Azure Subscriptions and Resource Groups provided for this training.

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
  key                 = "deploytomultiplesub.tfstate"
```

> Define your backend using the Storage Account you created few minutes ago.

#### Variables

In the *configuration/dev* folder, update the `dev.tfvars` file:

```hcl
resource_group_name = "the_name_of_your_main_resource_group"
```

### Exercise 2: Configure a new provider for the second targetted subscription (tagged as *feature*)

In the `provider.tf` file, add a new *provider* block, with the following configuration:

```hcl
provider "azurerm" {
  resource_provider_registrations = "none"
  features {}
  alias = "feature"
  subscription_id = var.feature_subscription_id
}
```

> Note the new `alias` attribute. This alias will be used later on within Terraform blocks to explicitly reference this provider.
> Note the `subscription_id` attribute. It allows terraform (when using this *feature* provider) to target a specific subscription.
> Using a variable, instead of a literal value, for the subscription gives freedom to make this value vary at Terraform run time.

In the `variables.tf` file, add the following block for the feature Subscription configuration:

```hcl
variable "feature_subscription_id" {
    type = string
    description = "Feature Subscription Id"
}
```

> No default value is provided for this variable ; value becomes mandatory at `terraform plan/apply` step!

in the `configuration/dev` folder add to the *dev.tfvars* file the variable value:

```hcl
feature_subscription_id = "Id_of_the_feature_subscription"
```

> Get the id from the feature subscription where is defined your resource group with tag `layer` = `feature`.

Reference your *feature* resource group adding a `data` block a `main.tf` file:

```hcl
data "azurerm_resource_group" "feature_rg" {
  name     = var.resource_group_name
  provider = azurerm.feature
}
```

> Note the `provider` attribute, used to target the *feature* subscription.

We are using a configuration where the name of the resource group is identical in  both **main** and **feature** subscriptions.

From the `src` folder, run the following commands to ensure both resource groups are available:

PowerShell

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
terraform init -backend-config="..\configuration\dev\backend.hcl"
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

Bash

```bash
az login
az account set --subscription "the_main_subscription_id"
export ARM_SUBSCRIPTION_ID="the_main_subscription_id"
terraform init -backend-config="..\configuration\dev\backend.hcl"
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

> At this point we have initialized Terraform backend in the *main* subscription, and we have configured Terraform template to use a resource group in the *feature* subscription. As this is a `data` block, no change is reported by Terraform.
>
> ![](../img/lab1-001.png)

### Exercise 3: Create the Virtual Networks

Add the following `resource` block in *main.tf* to create the Virtual Network in the *main* subscription:

```hcl
resource "azurerm_virtual_network" "main_vnet" {
  name                = "main-network"
  address_space       = ["10.0.0.0/24"]
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
}
```

> We do not explicitly set the provider. In that case Terraform uses the default one, which is the one without alias.
> This provider doesn't (re)define a subsription id and will target the *main* subscription you have set in your `az login` environment.

Run the following commands to deploy the Virtual Network:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

Add the following `resource` block in *main.tf* to create the Virtual Network in the *feature* subscription:

```hcl
resource "azurerm_virtual_network" "feature_vnet" {
  name                = "feature-network"
  address_space       = ["10.0.1.0/24"]
  location            = var.location
  resource_group_name = data.azurerm_resource_group.feature_rg.name
  provider            = azurerm.feature
}
```

> We explicitly set the provider targeting the feature subscription using its alias.

Run the following commands to deploy the Virtual Network:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal that both Virtual Networks have been created, one in the *main* subscription, and the second in the *feature* subscription.

### Exercise 4: Create peering

When done from the Azure portal, peering two VNets can be done from one or the other of the virtual networks.Within Terraform, this results into two peering actions, done at each sides of the link, so on the two VNet resources.

> We have to define two `resource` blocks for peering.

Add the following `resource` block to create the peering:

```hcl
resource "azurerm_virtual_network_peering" "peer-main-to-feature" {
  name                      = "peer-main-to-feature"
  resource_group_name       = data.azurerm_resource_group.self.name
  virtual_network_name      = azurerm_virtual_network.main_vnet.name
  remote_virtual_network_id = azurerm_virtual_network.feature_vnet.id
}

resource "azurerm_virtual_network_peering" "peer-feature-to-main" {
  name                      = "peer-feature-to-main"
  resource_group_name       = data.azurerm_resource_group.feature_rg.name
  virtual_network_name      = azurerm_virtual_network.feature_vnet.name
  remote_virtual_network_id = azurerm_virtual_network.main_vnet.id
  provider                  = azurerm.feature
}
```

> Note the provider attribute in the second peering block: this peering is starting from the *feature* VNet and goes to the *main* VNet.

Run the following commands to deploy the peering:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check the Network peering in the Azure Portal.
>
> Using providers in Terraform template allows you to target different subscriptions.
>
> During this lab we have been using same identity and credentials to reach the different subscriptions (authentication set via the Azure CLI with `az login`), but this can also vary.
> To use different identities per provider you shall define them in each `provider` block as (e.g. if using SP with client id/client secret):
>
> Default provider
>
> ```hcl
> provider "azurerm" {
>  features {}
>  subscription_id = var.main_subscription_id
>  client_id       = var.main_client_id
>  client_secret   = var.main_client_secret
>  tenant_id       = var.tenant_id
> }
> ```
>
> Feature provider
>
> ```hcl
> provider "azurerm" {
>  alias = "feature"
>  features {}
>  subscription_id = var.feature_subscription_id
>  client_id       = var.feature_client_id
>  client_secret   = var.feature_client_secret
>  tenant_id       = var.tenant_id
> }
> ```

### Exercise 5: Remove resources

Destroy the created resources

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Destroying resources cleans the *tfstate* (along with resources in Azure) but not the template files.
> Resources will be recreated at next `terraform apply` time.
