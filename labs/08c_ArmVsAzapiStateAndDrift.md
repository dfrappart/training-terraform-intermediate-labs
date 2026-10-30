# ARM template vs azapi: state, plan and drift

- [ARM template vs azapi: state, plan and drift](#arm-template-vs-azapi-state-plan-and-drift)
  - [Lab overview](#lab-overview)
  - [Objectives](#objectives)
  - [Instructions](#instructions)
    - [Before you start](#before-you-start)
    - [Exercise 1: Setup your environment](#exercise-1-setup-your-environment)
    - [Exercise 2: Declare the same Virtual Network twice](#exercise-2-declare-the-same-virtual-network-twice)
    - [Exercise 3: Read the plan](#exercise-3-read-the-plan)
    - [Exercise 4: Look at the state](#exercise-4-look-at-the-state)
    - [Exercise 5: Reference the resources](#exercise-5-reference-the-resources)
    - [Exercise 6: Introduce drift](#exercise-6-introduce-drift)
    - [Exercise 7: Remove Resources](#exercise-7-remove-resources)

## Lab overview

Lab 8 deployed a Virtual Network with an ARM template wrapped in `azurerm_resource_group_template_deployment`.
That resource makes any Azure feature reachable from Terraform, whatever the azurerm version. The **azapi** provider
offers the same reach with a different design: each Azure resource is a Terraform resource of its own.

Being able to deploy something is only half of what Terraform provides. The other half is **knowing**: what is deployed,
what a change will do, and whether reality still matches the code. This lab deploys the *same* Virtual Network both ways
and compares what Terraform knows afterwards.

> The previous lab, *08b*, pins azurerm to an old version to show which features it cannot reach. This lab does not.
> Both resources work with current provider versions; what differs is the quality of the information Terraform keeps.

## Objectives

After you complete this lab, you will be able to:

- Deploy the same Virtual Network with an ARM template and with azapi
- Compare the plan, the state content and the references of both approaches
- Observe how each approach reacts to a change made outside Terraform
- Explain why an unnoticed difference between code and reality is a risk

## Instructions

### Before you start

- Ensure Terraform is installed and available from the system's PATH. The training setup pins `~> 1.13.0`; this lab relaxes
  it to `>= 1.13.0` so that newer 1.x versions work.
- Ensure Azure CLI is installed and you are logged in.
- Check your access to the Azure Subscription and Resource Group provided for this training.

> This lab was written and checked with **Terraform 1.17.0, azurerm 5.12.0 and azapi 2.13.0**. Results in other versions may differ,
> so note your own versions with `terraform version` when you report your observations.

### Exercise 1: Setup your environment

In your *main* Resource Group (the one tagged with attribute **layer** with value **main**), create a Storage Account, with a Blob container named **tfstate**

Clone the repository <https://github.com/smartinez-cellenza/training-terraform-intermediate-labs-setup>

```bash
git clone https://github.com/smartinez-cellenza/training-terraform-intermediate-labs-setup.git
cd training-terraform-intermediate-labs-setup
```

> This template contains a basic Terraform project configuration

In the *configuration/dev* folder, update the **backend.hcl** file:

- **resource_group_name**  = "the_name_of_the_resource_group_of_your_state_storage"
- **storage_account_name** = "the_name_of_your_state_storage_account"
- **container_name**       = "tfstate"
- **key**                  = "lab08c.tfstate"

In the *configuration/dev* folder, update the *dev.tfvars* file:

- **resource_group_name** = "the_name_of_your_main_resource_group"

Replace the content of **src/version.tf** with:

```hcl
terraform {
  required_version = ">= 1.13.0"

  backend "azurerm" {}

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.0"
    }
    azapi = {
      source  = "Azure/azapi"
      version = "~> 2.0"
    }
  }
}
```

> The azapi provider needs no `provider` block here: it reads the same `ARM_SUBSCRIPTION_ID` and Azure CLI login as azurerm.

Create a **src/locals.tf** file:

```hcl
locals {
  vnet_name_arm   = "vnet-lab08c-arm"
  vnet_name_azapi = "vnet-lab08c-azapi"
}
```

Initialize:

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
cd src
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform version
```

### Exercise 2: Declare the same Virtual Network twice

Both Virtual Networks get the same settings: an address space, a tag and the `privateEndpointVNetPolicies` property set to `Basic`
(API version `2024-05-01`). Only the address space differs, so that the two networks do not overlap.

#### The ARM template

Create the folder **src/templates** and the file **src/templates/vnet.json**:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vnetName": {
      "type": "String",
      "metadata": {
        "description": "Name of the VNET"
      }
    }
  },
  "resources": [
    {
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2024-05-01",
      "name": "[parameters('vnetName')]",
      "location": "[resourceGroup().location]",
      "tags": {
        "approach": "arm-template"
      },
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "10.10.0.0/16"
          ]
        },
        "privateEndpointVNetPolicies": "Basic"
      }
    }
  ],
  "outputs": {
    "vnetId": {
      "type": "String",
      "value": "[resourceId('Microsoft.Network/virtualNetworks', parameters('vnetName'))]"
    }
  }
}
```

Create **src/arm_deployment.tf**:

```hcl
# Example 1: the virtual network is deployed by an ARM template wrapped in a Terraform resource
resource "azurerm_resource_group_template_deployment" "vnet" {
  name                = "deploy-vnet-arm"
  resource_group_name = data.azurerm_resource_group.self.name
  deployment_mode     = "Incremental"

  parameters_content = jsonencode({
    vnetName = {
      value = local.vnet_name_arm
    }
  })
  template_content = file("${path.module}/templates/vnet.json")
}
```

#### The azapi resource

Create **src/azapi_vnet.tf**:

```hcl
# Example 2: the same virtual network, declared as a resource of its own through azapi
resource "azapi_resource" "vnet" {
  type      = "Microsoft.Network/virtualNetworks@2024-05-01"
  name      = local.vnet_name_azapi
  parent_id = data.azurerm_resource_group.self.id
  location  = data.azurerm_resource_group.self.location

  tags = {
    approach = "azapi"
  }

  body = {
    properties = {
      addressSpace = {
        addressPrefixes = ["10.20.0.0/16"]
      }
      privateEndpointVNetPolicies = "Basic"
    }
  }

  response_export_values = ["properties.privateEndpointVNetPolicies"]
}
```

> - `type` is the ARM resource type, plus the API version after `@`.
> - `parent_id` is the Resource ID of the scope. For a resource in a Resource Group, it is the ID of the Resource Group.
> - `body` is the same JSON that you would send to the REST API, written as an HCL object.
> - `response_export_values` lists values of the API response that Terraform exposes in the `output` attribute of the resource.

Create **src/outputs.tf**:

```hcl
output "arm_vnet_id" {
  value = jsondecode(azurerm_resource_group_template_deployment.vnet.output_content).vnetId.value
}

output "azapi_vnet_id" {
  value = azapi_resource.vnet.id
}

output "azapi_vnet_policy" {
  value = azapi_resource.vnet.output.properties.privateEndpointVNetPolicies
}
```

### Exercise 3: Read the plan

Do not apply yet. Run:

```powershell
terraform fmt
terraform validate
terraform plan -var-file="..\configuration\dev\dev.tfvars"
```

Compare how each resource is described. Excerpts of what the plan shows (values depend on your configuration):

```text
  # azapi_resource.vnet will be created
  + resource "azapi_resource" "vnet" {
      + body                      = {
          + properties = {
              + addressSpace                = {
                  + addressPrefixes = [
                      + "10.20.0.0/16",
                    ]
                }
              + privateEndpointVNetPolicies = "Basic"
            }
        }
      ...

  # azurerm_resource_group_template_deployment.vnet will be created
  + resource "azurerm_resource_group_template_deployment" "vnet" {
      ...
      + template_content    = jsonencode(
            {
              + "$schema"      = "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#"
              + contentVersion = "1.0.0.0"
              + outputs        = { ... }
              + parameters     = { ... }
              + resources      = [
                  + {
                      + apiVersion = "2024-05-01"
                      ...
                      + type       = "Microsoft.Network/virtualNetworks"
                    },
                ]
            }
        )
    }
```

> **Questions:**
>
> - In the plan, which one shows a resource called "virtual network"?
> - If you changed the address space later, how would each plan display the change?

Apply:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

### Exercise 4: Look at the state

```powershell
terraform state list
terraform state show azurerm_resource_group_template_deployment.vnet
terraform state show azapi_resource.vnet
az network vnet list --resource-group "the_name_of_your_main_resource_group" --query "[].{name:name, prefixes:addressSpace.addressPrefixes, policy:privateEndpointVNetPolicies, tags:tags}" -o json
```

> **Questions:**
>
> - Both networks exist in Azure. Which one does Terraform know as a resource?
> - What does Terraform know about the Virtual Network created by the ARM template?

### Exercise 5: Reference the resources

Look at the outputs:

```powershell
terraform output
```

> **Questions:**
>
> - How did you get the ID of each Virtual Network? Compare the expressions in **outputs.tf**.
> - Imagine a Subnet or a Network Security Group that must be attached to each network. What would you reference in each case?
> - What does `response_export_values` give you that the ARM template output does not?

### Exercise 6: Introduce drift

*Drift* is a difference between your code and the real infrastructure, introduced outside Terraform. A tool that manages
infrastructure as code should tell you about it in `plan`.

Check that both approaches are in sync:

```powershell
terraform plan -var-file="..\configuration\dev\dev.tfvars"
```

The plan should say `No changes`. Now change both Virtual Networks with the Azure CLI. The values below are examples;
any address space that differs from the code will do.

```powershell
az network vnet update --resource-group "the_name_of_your_main_resource_group" --name vnet-lab08c-arm   --address-prefixes 40.1.0.0/24
az network vnet update --resource-group "the_name_of_your_main_resource_group" --name vnet-lab08c-azapi --address-prefixes 40.2.0.0/24
az network vnet update --resource-group "the_name_of_your_main_resource_group" --name vnet-lab08c-azapi --set tags.environment=drift
terraform plan -var-file="..\configuration\dev\dev.tfvars"
```

Apply, then check the address spaces in Azure:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
az network vnet list --resource-group "the_name_of_your_main_resource_group" --query "[].{name:name, prefixes:addressSpace.addressPrefixes}" -o table
```

> **Questions:**
>
> - Why does the deployment resource not see the change? What does it compare, and what does it not compare?
> - Terraform says `No changes` while the real Virtual Network differs from the code. What can happen the next time somebody trusts that plan?
> - Which approach would you prefer for a resource that other teams can modify in the portal?

### Exercise 7: Remove Resources

First, check what destroying the ARM template deployment really removes. Destroy only that resource:

```powershell
terraform destroy -target=azurerm_resource_group_template_deployment.vnet -var-file="..\configuration\dev\dev.tfvars"
az network vnet list --resource-group "the_name_of_your_main_resource_group" --query "[].name" -o tsv
```

> **Question:** is `vnet-lab08c-arm` still there? Look up the `template_deployment` setting in the `features` block of the azurerm
> provider and explain what you observe.

Then remove everything else:

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars"
az network vnet list --resource-group "the_name_of_your_main_resource_group" --query "[].name" -o tsv
```

> If a Virtual Network created by the ARM template is still listed, delete it with `az network vnet delete`.
