# azurerm vs azapi: exploring the limits of a provider

- [azurerm vs azapi: exploring the limits of a provider](#azurerm-vs-azapi-exploring-the-limits-of-a-provider)
  - [Lab overview](#lab-overview)
  - [Objectives](#objectives)
  - [Instructions](#instructions)
    - [Before you start](#before-you-start)
    - [Exercise 1: Setup your environment](#exercise-1-setup-your-environment)
    - [Exercise 2: Baseline, a Virtual Network through an ARM template](#exercise-2-baseline-a-virtual-network-through-an-arm-template)
    - [Exercise 3: Hit the limit of azurerm (a missing property)](#exercise-3-hit-the-limit-of-azurerm-a-missing-property)
    - [Exercise 4: Solve it, two ways](#exercise-4-solve-it-two-ways)
    - [Exercise 5: Hit the limit of azurerm (a missing resource)](#exercise-5-hit-the-limit-of-azurerm-a-missing-resource)
    - [Exercise 6: The limits of azapi](#exercise-6-the-limits-of-azapi)
    - [Exercise 7: Remove Resources](#exercise-7-remove-resources)

## Lab overview

Lab 8 deployed a Virtual Network with an ARM template, wrapped in the `azurerm_resource_group_template_deployment` resource.
This lab goes further. It asks **why** you would ever need that work-around, and whether the **azapi** provider is a better one.

- **azurerm** exposes each Azure resource as a hand-written, typed Terraform resource. Each resource is built on one specific
  version of the Azure REST API. A property or a resource that Azure supports does not exist in Terraform until someone adds it
  to the provider and a new version is released.
- **azapi** is a thin layer over the Azure Resource Manager (ARM) REST API. You give it a resource type, an API version and a
  JSON-like `body`. Any resource type and API version that Azure supports can be used the day it is published.

To make the difference visible, this lab **pins azurerm to the version used in lab 8 (`~> 4.0.0`)**. Features that were added to azurerm
later are, for this lab, out of reach for azurerm. You will confirm that, then deploy them with azapi.

> Drift detection is compared in depth in lab *08c*.

## Objectives

After you complete this lab, you will be able to:

- Identify a feature that Azure supports but that a given azurerm version does not
- Deploy that feature with an ARM template wrapped in Terraform, and with azapi
- Compare state content, plan readability and destroy behaviour between the approaches
- Explain the limits and risks of azapi

## Instructions

### Before you start

- Ensure Terraform is installed and available from the system's PATH. The training setup pins `~> 1.13.0`; this lab relaxes
  it to `>= 1.13.0` so that newer 1.x versions work.
- Ensure Azure CLI is installed and you are logged in.
- Check your access to an Azure Subscription where you can create a Resource Group.

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
- **key**                  = "azapi.tfstate"

> This template use Partial backend configuration. The Storage Account just created is used for tfstate file persistence

In the *configuration/dev* folder, update the *dev.tfvars* file :

- **resource_group_name** = "the_name_of_your_main_resource_group"

> The setup uses `northeurope` by default. Virtual networks work in any region. Region choice matters later for VM sizes,
> but not for this lab. Pick a region that supports Network Security Perimeters (Exercise 5); `swedencentral` for example does.

Replace the content of **src/version.tf** with:

```hcl
terraform {
  required_version = ">= 1.13.0"

  backend "azurerm" {}

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0.0"
    }
    azapi = {
      source  = "Azure/azapi"
      version = "~> 2.0"
    }
  }
}
```

> `~> 4.0.0` means: any 4.0.x version, and nothing from 4.1 onwards. This is the deliberate limitation of the lab.

The azapi provider needs no `provider` block here: it reads the same `ARM_SUBSCRIPTION_ID` and Azure CLI login as azurerm.

Create a **src/locals.tf** file:

```hcl
locals {
  vnet_name       = "vnet-lab08b-arm"
  vnet_name_azapi = "vnet-lab08b-azapi"
}
```

Initialize and check the installed versions:

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
cd src
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform version
```

> **Question:** which azurerm version did Terraform install? Is it the latest one published in the registry? Why not?

### Exercise 2: Baseline, a Virtual Network through an ARM template

Create **src/arm_deployment.tf** with the lab 8 deployment, using the names defined above:

```hcl
resource "azurerm_resource_group_template_deployment" "example" {
  name                = "example-deploy"
  resource_group_name = data.azurerm_resource_group.self.name
  deployment_mode     = "Incremental"
  parameters_content = jsonencode({
    "vnetName" = {
      value = local.vnet_name
    }
  })
  template_content = <<TEMPLATE
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
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Network/virtualNetworks",
            "apiVersion": "2020-05-01",
            "name": "[parameters('vnetName')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "addressSpace": {
                    "addressPrefixes": [
                        "10.0.0.0/16"
                    ]
                }
            }
        }
    ],
    "outputs": {
      "exampleOutput": {
        "type": "String",
        "value": "someoutput"
      }
    }
}
TEMPLATE
}

output "arm_example_output" {
  value = jsondecode(azurerm_resource_group_template_deployment.example.output_content).exampleOutput.value
}
```

Deploy it:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

**Observe and record:**

```powershell
terraform state list
terraform state show azurerm_resource_group_template_deployment.example
az network vnet show --resource-group "the_name_of_your_lab_resource_group" --name vnet-lab08b-arm --query "{name:name, location:location, addressSpace:addressSpace.addressPrefixes}"
```

### Exercise 3: Hit the limit of azurerm (a missing property)

Azure added a `privateEndpointVNetPolicies` property to Virtual Networks. It enables network policies for private endpoints
at the VNet level. In azurerm it became the argument `private_endpoint_vnet_policies`, **added in provider version 4.15.0**
(pull request [#27830](https://github.com/hashicorp/terraform-provider-azurerm/issues/27830), merged 8 January 2025).

Create **src/limit_property.tf**:

```hcl
resource "azurerm_virtual_network" "limit" {
  name                = "vnet-lab08b-limit"
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
  address_space       = ["10.2.0.0/16"]

  private_endpoint_vnet_policies = "Basic"
}
```

```powershell
terraform validate
```

Result observed with azurerm 4.0.1:

```text
Error: Unsupported argument

  on limit_property.tf line 11, in resource "azurerm_virtual_network" "limit":
  11:   private_endpoint_vnet_policies = "Basic"

An argument named "private_endpoint_vnet_policies" is not expected here.
```

> **Questions:**
>
> - The feature exists in Azure. Why can Terraform not use it?
> - Compare the azurerm documentation of the 4.0.x version with the latest one for `azurerm_virtual_network`.
> - Terraform and azurerm are updated frequently. Is a frequent release cycle the same thing as being up to date with the Azure API?

**Delete `limit_property.tf`** before continuing, otherwise `validate`, `plan` and `apply` keep failing.

### Exercise 4: Solve it, two ways

#### Option A: the ARM template (the lab 8 approach)

The ARM template is not limited by the azurerm version, because Azure itself reads the template. Update the template in
**arm_deployment.tf**: set the `apiVersion` of the VNet to `2024-03-01` and add the property.

```json
"apiVersion": "2024-03-01",
...
"properties": {
    "addressSpace": {
        "addressPrefixes": [
            "10.0.0.0/16"
        ]
    },
    "privateEndpointVNetPolicies": "Basic"
}
```

```powershell
terraform plan -var-file="..\configuration\dev\dev.tfvars"
terraform apply -var-file="..\configuration\dev\dev.tfvars"
az network vnet show --resource-group "the_name_of_your_lab_resource_group" --name vnet-lab08b-arm --query "privateEndpointVNetPolicies"
```

> **Question:** what does the plan show? Do you see the property change, or a change inside a JSON string?

#### Option B: azapi

Create **src/azapi_vnet.tf**:

```hcl
resource "azapi_resource" "vnet" {
  type      = "Microsoft.Network/virtualNetworks@2024-03-01"
  name      = local.vnet_name_azapi
  parent_id = data.azurerm_resource_group.self.id
  location  = var.location

  body = {
    properties = {
      addressSpace = {
        addressPrefixes = ["10.1.0.0/16"]
      }
      privateEndpointVNetPolicies = "Basic"
    }
  }
}

output "azapi_vnet_id" {
  value = azapi_resource.vnet.id
}
```

> - `type` is the ARM resource type, plus the API version after `@`.
> - `parent_id` is the Resource ID of the scope. For a resource in a Resource Group, it is the ID of the Resource Group;
>   the azurerm data source already exposes it as `.id`.
> - `body` is the same JSON that you would send to the REST API, written as an HCL object.
> - Unlike Bicep, azapi **can** take the API version from an expression, for example
>   `"Microsoft.Network/virtualNetworks@${var.api_version}"`.

```powershell
terraform plan -var-file="..\configuration\dev\dev.tfvars"
terraform apply -var-file="..\configuration\dev\dev.tfvars"
az network vnet show --resource-group "the_name_of_your_lab_resource_group" --name vnet-lab08b-azapi --query "privateEndpointVNetPolicies"
terraform state list
terraform state show azapi_resource.vnet
```

> **Questions:**
>
> - Which API version accepts the property? The oldest one in which we found it is `2024-03-01`
>   (`2024-01-01` rejects it, see Exercise 6). How would you find that version for another property?
> - Compare `terraform state list` and `terraform state show` with Exercise 2. What does Terraform now know?
> - Compare how the plan reads in Option A and Option B.

### Exercise 5: Hit the limit of azurerm (a missing resource)

Network Security Perimeters (NSP) are a newer Azure feature. In azurerm, `azurerm_network_security_perimeter` and its related
resources were **added in provider version 4.58.0**.

Create **src/limit_resource.tf**:

```hcl
resource "azurerm_network_security_perimeter" "limit" {
  name                = "nsp-lab08b"
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
}
```

```powershell
terraform validate
```

Result observed with azurerm 4.0.1:

```text
Error: Invalid resource type

  on limit_resource.tf line 1, in resource "azurerm_network_security_perimeter" "limit":
   1: resource "azurerm_network_security_perimeter" "limit" {

The provider hashicorp/azurerm does not support resource type
"azurerm_network_security_perimeter".
```

This time the *whole resource type* is missing, not only a property. **Delete `limit_resource.tf`**, then create **src/azapi_nsp.tf**:

```hcl
resource "azapi_resource" "nsp" {
  type      = "Microsoft.Network/networkSecurityPerimeters@2024-07-01"
  name      = "nsp-lab08b"
  parent_id = data.azurerm_resource_group.self.id
  location  = var.location
  body      = {}
}
```

> The GA API versions of `Microsoft.Network/networkSecurityPerimeters` start at `2024-07-01`. You can list the versions and
> regions your subscription sees with:
>
> ```powershell
> az provider show --namespace Microsoft.Network --query "resourceTypes[?resourceType=='networkSecurityPerimeters'].apiVersions[]" -o tsv
> ```

```powershell
terraform plan -var-file="..\configuration\dev\dev.tfvars"
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

> **Questions:**
>
> - What is the difference between a missing property (Exercise 3) and a missing resource type, in terms of the workarounds available?
> - How would you reference the perimeter from another resource, since there are no typed attributes?

### Exercise 6: The limits of azapi

azapi removes the limits of the provider, but it has costs of its own. Explore them.

**1. API version ownership.** In **azapi_vnet.tf**, change the version to `2024-01-01` and run `terraform plan`.

```text
Error: Invalid configuration
embedded schema validation failed: the argument "body" is invalid:
`properties.privateEndpointVNetPolicies` is not expected here. ...
You can try to update `azapi` provider to the latest version or disable the validation using the feature flag
`schema_validation_enabled = false` within the resource block
```

**2. Schema validation.** Restore `2024-03-01`, then introduce a typo in a property name (for example `privateEndpointVNetPolicy`) and run `terraform plan`.
Then add `schema_validation_enabled = false` to the resource and run the plan again.

> **Questions:**
>
> - What does the schema validation catch? What happens when it is disabled?
> - Who is responsible for picking the API version, and for upgrading it later?
> - Does changing the API version change the behaviour of the resource, even when the `body` does not change?

**3. Other points to document**, using the azapi and Azure documentation:

- The `body` is not typed: no autocompletion, no per-argument documentation in the provider.
- The state contains what the API returns. Check for secrets with `terraform state show`.
- azurerm and azapi can be mixed in the same configuration (this lab does it: the Resource Group comes from an azurerm data source).
- Other azapi resources: `azapi_update_resource` (change a few properties of a resource created elsewhere), `azapi_resource_action`
  (call an action, for example a `POST` operation), `azapi_resource_list` and the data-plane resources. Find one problem
  they solve that azurerm cannot.

### Exercise 7: Remove Resources

First, check what destroying the ARM template deployment really removes. Destroy only that resource:

```powershell
terraform destroy -target=azurerm_resource_group_template_deployment.example -var-file="..\configuration\dev\dev.tfvars"
az network vnet list --resource-group "the_name_of_your_lab_resource_group" --query "[].name" -o tsv
```

> **Question:** is `vnet-lab08b-arm` still there? Look up the `template_deployment` setting in the `features` block of the
> azurerm provider and explain what you observe.

Then remove everything else:

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars"
az resource list --resource-group "the_name_of_your_lab_resource_group" -o table
```

> The lab Resource Group and the state storage were created outside Terraform. Delete the lab Resource Group yourself when you are done:
>
> ```powershell
> az group delete --name "the_name_of_your_lab_resource_group"
> ```
