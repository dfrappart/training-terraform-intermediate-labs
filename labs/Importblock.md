# Setup environment

## Lab overview

In this lab, you will continue your experimentations on resources import, this time using the import block.
You will also have a look at code refactoring with the moved block.

## Objectives

After you complete this lab, you will be able to:

-   Import resources using the import block
-   Use terraform feature for generating automatically resources configuration
-   Refactor code using the moved block

## Instructions

### Before you start

- Ensure Terraform (version >= 1.5.0) is installed and available from system's PATH.
- Ensure Azure CLI is installed.
- Check your access to the Azure Subscriptions and Resource Groups provided for this training.

### Exercise 1: Setup your environment

In your *main* Resource Group (the one tagged with attribute **layer** with value **main**), create a Storage Account, with a Blob container named **tfstate**

Clone the repository https://gitvnet_name.com/smartinez-cellenza/training-terraform-intermediate-labs-setup

```bash
git clone https://gitvnet_name.com/smartinez-cellenza/training-terraform-intermediate-labs-setup.git
cd training-terraform-intermediate-labs-setup
```

> This configuration contains a basic Terraform project configuration

In the *configuration/dev* folder, update the **backend.hcl** file :

- **resource_group_name**  = "the_name_of_your_main_resource_group"
- **storage_account_name** = "the_name_of_the_storage_account_you_just_created"
- **container_name**       = "tfstates"
- **key**                  = "import.tfstate"

> This configuration use Partial backend configuration. The Storage Account just created is used for tfstate file persistence

In the *configuration/dev* folder, update the *dev.tfvars* file :

- **resource_group_name** = "the_name_of_your_main_resource_group"

### Exercise 2: Create a Virtual Network with terraform and add subnets from the portal

Add the following to your `variable.tf` file: 

```go

variable "vnet_nameconfig" {
  type = object({
    address_space = optional(list(string), ["10.100.0.0/24"])
    location = optional(string, "eastus")
    vnet_namename = optional(string,"vnet_name")
  })

```
And specify the value in the dev.tfvars file: 

```go

vnet_nameconfig = {
    address_space = ["10.0.0.0/24"]
}

```

Add the following to your `main.tf` file:

```go

resource "azurerm_virtual_network" "main_vnet" {
  name                = var.vnet_nameconfig.vnet_namename
  address_space       = var.vnet_nameconfig.address_space
  location            = var.vnet_nameconfig.location
  resource_group_name = data.azurerm_resource_group.self.name
}

```

Create the vnet with a `terraform apply` command

```bash
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
```

You can check the existence of your virtual network with the command `az network`:

```bash

yumemaru [ ~ ]$ az network vnet show -n vnet_name -g rsg_name
{
  "addressSpace": {
    "addressPrefixes": [
      "10.0.0.0/24"
    ]
  },
  "dhcpOptions": {
    "dnsServers": []
  },
  "enableDdosProtection": false,
  "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name",
  "location": "eastus",
  "name": "vnet_name",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg_name",
  "resourceGuid": "00000000-0000-0000-0000-000000000000",
  "subnets": [],
  "tags": {},
  "type": "Microsoft.Network/virtualNetworks",
  "virtualNetworkPeerings": []
}

```

Create 4 subnets on your virtual network with the command `az network vnet create -n subnet_name -g rsg_name --vnet-name vnet_name --address-prefixes address_prefix_for_subnet`.
**Note: To create 4 subnets on the virtual network range, you can use `/26` cidr**

```bash

yumemaru [ ~ ]$ az network vnet subnet create -g rsg_name --vnet-name vnet_name --name subnet1 --address-prefixes 10.0.0.0/26
{
  "addressPrefix": "10.0.0.0/26",
  "delegations": [],
  "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet1",
  "name": "subnet1",
  "privateEndpointNetworkPolicies": "Disabled",
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg_name",
  "type": "Microsoft.Network/virtualNetworks/subnets"
}
yumemaru [ ~ ]$ az network vnet subnet create -g rsg_name --vnet-name vnet_name --name subnet2 --address-prefixes 10.0.0.64/26
{
  "addressPrefix": "10.0.0.64/26",
  "delegations": [],
  "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet2",
  "name": "subnet2",
  "privateEndpointNetworkPolicies": "Disabled",
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg_name",
  "type": "Microsoft.Network/virtualNetworks/subnets"
}
yumemaru [ ~ ]$ az network vnet subnet create -g rsg_name --vnet-name vnet_name --name subnet3 --address-prefixes 10.0.0.128/26
{
  "addressPrefix": "10.0.0.128/26",
  "delegations": [],
  "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet3",
  "name": "subnet3",
  "privateEndpointNetworkPolicies": "Disabled",
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg_name",
  "type": "Microsoft.Network/virtualNetworks/subnets"
}
yumemaru [ ~ ]$ az network vnet subnet create -g rsg_name --vnet-name vnet_name --name AzureBastionSubnet --address-prefixes 10.0.0.192/26
{
  "addressPrefix": "10.0.0.192/26",
  "delegations": [],
  "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
  "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/AzureBastionSubnet",
  "name": "AzureBastionSubnet",
  "privateEndpointNetworkPolicies": "Disabled",
  "privateLinkServiceNetworkPolicies": "Enabled",
  "provisioningState": "Succeeded",
  "resourceGroup": "rsg_name",
  "type": "Microsoft.Network/virtualNetworks/subnets"
}

```

Check again the virtual network with the command `az network vnet show -n vnet_name -g rsg_name`. You should see the subnets in the `subnets` section: 

```json

"subnets": [
    {
      "addressPrefix": "10.0.0.0/26",
      "delegations": [],
      "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet1",
      "name": "subnet1",
      "privateEndpointNetworkPolicies": "Disabled",
      "privateLinkServiceNetworkPolicies": "Enabled",
      "provisioningState": "Succeeded",
      "resourceGroup": "rsg_name",
      "type": "Microsoft.Network/virtualNetworks/subnets"
    },
    {
      "addressPrefix": "10.0.0.64/26",
      "delegations": [],
      "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet2",
      "name": "subnet2",
      "privateEndpointNetworkPolicies": "Disabled",
      "privateLinkServiceNetworkPolicies": "Enabled",
      "provisioningState": "Succeeded",
      "resourceGroup": "rsg_name",
      "type": "Microsoft.Network/virtualNetworks/subnets"
    },
    {
      "addressPrefix": "10.0.0.128/26",
      "delegations": [],
      "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet3",
      "name": "subnet3",
      "privateEndpointNetworkPolicies": "Disabled",
      "privateLinkServiceNetworkPolicies": "Enabled",
      "provisioningState": "Succeeded",
      "resourceGroup": "rsg_name",
      "type": "Microsoft.Network/virtualNetworks/subnets"
    },
    {
      "addressPrefix": "10.0.0.192/26",
      "delegations": [],
      "etag": "W/\"00000000-0000-0000-0000-000000000000\"",
      "id": "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/AzureBastionSubnet",
      "name": "AzureBastionSubnet",
      "privateEndpointNetworkPolicies": "Disabled",
      "privateLinkServiceNetworkPolicies": "Enabled",
      "provisioningState": "Succeeded",
      "resourceGroup": "rsg_name",
      "type": "Microsoft.Network/virtualNetworks/subnets"
    }
  ]

```

Check the state to see the vnet object with the command `terraform state show`:

```bash

df@df2204lts:~/Documents/myrepo/tf-intermediaire-lab2/src$ terraform state show azurerm_virtual_network.main_vnet
# azurerm_virtual_network.main_vnet:
resource "azurerm_virtual_network" "main_vnet" {
    address_space           = [
        "10.0.0.0/24",
    ]
    dns_servers             = []
    flow_timeout_in_minutes = 0
    guid                    = "00000000-0000-0000-0000-000000000000"
    id                      = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name"
    location                = "eastus"
    name                    = "vnet_name"
    resource_group_name     = "rsg_name"
    subnet                  = []
}

```

The result should display an empty list of subnets, because those are not currrently managed by terraform.

### Exercise 3: Import 1 subnet with an import block

To add the subnets in the terraform coonfiguration, we will use the terraform import block on the first subnet.

This import block is written as this:

```go

import {
    to = <resource_address_in_tf_config>
    id = <resource_id_in_the_cloud_provider>
}

```

Create a new file imports.tf and add the following block to import your first subnet:

```go

import {
    to = azurerm_subnet.subnet1
    id = "subnet_resource_id"
}

```

Create a new file subnets.tf and add the following configuration to match the imported resource:

```go

resource "azurerm_subnet" "subnet1" {
  name = "subnet1"
  resource_group_name = data.azurerm_resource_group.self.name
  virtual_network_name = azurerm_virtual_network.main_vnet.name
  address_prefixes = ["10.0.0.0/26"]
  
}
```

```bash

yumemaru [ ~ ]$ terraform plan -var-file="../configuration/dev/dev.tfvars"
data.azurerm_resource_group.feature_rg: Reading...
data.azurerm_resource_group.main: Reading...
data.azurerm_resource_group.feature_rg: Read complete after 1s [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg_name-2]
azurerm_virtual_network.feature_vnet["vnet2"]: Refreshing state... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg_name-2/===============================================truncated/===============================================

Terraform will perform the following actions:

  # azurerm_subnet.subnet1 will be imported
    resource "azurerm_subnet" "subnet1" {
        address_prefixes                               = [
            "10.0.0.0/26",
        ]
        enforce_private_link_endpoint_network_policies = true
        enforce_private_link_service_network_policies  = false
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet1"
        name                                           = "subnet1"
        private_endpoint_network_policies_enabled      = false
        private_link_service_network_policies_enabled  = true
        resource_group_name                            = "rg_name"
        service_endpoint_policy_ids                    = []
        service_endpoints                              = []
        virtual_network_name                           = "vnet_name"
    }

Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

```

The plan display no resource to create or modify, but there is a `1 to import mention`.
Run a terraform apply to validate the import:


```bash

Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

azurerm_subnet.subnet1: Importing... [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet1]
azurerm_subnet.subnet1: Import complete [id=/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet1]

Apply complete! Resources: 1 imported, 0 added, 0 changed, 0 destroyed.

```

### Exercise 4: Import the other subnets with import blocks and generate automatically the terraform configuration

To import th other subnets, add the following import blocks in the `imports.tf` file:

```go

import {
    to = azurerm_subnet.subnet2
    id = "subnet2_resource_id"
}


import {
    to = azurerm_subnet.subnet3
    id = "subnet3_resource_id"
}

import {
    to = azurerm_subnet.AzureBastionSubnet
    id = "AzureBastionSubnet_resource_id"
}
```

Run a `terraform plan` command with the appropriate argument. You should get an error for each subnet to be imported. It's an expected error because we did not write the subnets terraform configuration.

```bash

╷
│ Error: Configuration for import target does not exist
│ 
│   on imports.tf line 21, in import:
│   21:     to = azurerm_subnet.AzureBastionSubnet
│ 
│ The configuration for the given import target azurerm_subnet.AzureBastionSubnet does not exist. If you wish to automatically generate config for this resource,
│ use the -generate-config-out option within terraform plan. Otherwise, make sure the target resource exists within your configuration. For example:
│ 
│   terraform plan -generate-config-out=generated.tf
╵
```

Notice, at the end of the error, the suggestion to use `terraform plan -generate-config-out=generated.tf`.

Relaunch a `terraform plan` with the suggested additional argument.

You may get the folllowing error:

```bash

Planning failed. Terraform encountered an error while generating this plan.

╷
│ Error: Not enough list items
│ 
│   with azurerm_subnet.subnet2,
│   on generated.tf line 6:
│   (source code not available)
│ 
│ Attribute service_endpoint_policy_ids requires 1 item minimum, but config has only 0 declared.

```

Ignore this error for now, an check in your folder the existence of a `generated.tf` file.

It should contains something similar to this:

```go

# __generated__ by Terraform
# Please review these resources and move them into your main configuration files.

# __generated__ by Terraform
resource "azurerm_subnet" "AzureBastionSubnet" {
  address_prefixes                              = ["10.0.0.192/26"]
  name                                          = "AzureBastionSubnet"
  private_endpoint_network_policies_enabled     = false
  private_link_service_network_policies_enabled = true
  resource_group_name                           = "rg_name"
  service_endpoint_policy_ids                   = []
  service_endpoints                             = []
  virtual_network_name                          = "vnet_name"
}

# __generated__ by Terraform
resource "azurerm_subnet" "subnet3" {
  address_prefixes                              = ["10.0.0.128/26"]
  name                                          = "subnet3"
  private_endpoint_network_policies_enabled     = false
  private_link_service_network_policies_enabled = true
  resource_group_name                           = "rg_name"
  service_endpoint_policy_ids                   = []
  service_endpoints                             = []
  virtual_network_name                          = "vnet_name"
}

# __generated__ by Terraform
resource "azurerm_subnet" "subnet2" {
  address_prefixes                              = ["10.0.0.64/26"]
  name                                          = "subnet2"
  private_endpoint_network_policies_enabled     = false
  private_link_service_network_policies_enabled = true
  resource_group_name                           = "rg_name"
  service_endpoint_policy_ids                   = []
  service_endpoints                             = []
  virtual_network_name                          = "vnet_name"
}

```

Comment the line for `service_endpoints` and `service_endpoint_policy_ids` as those were the parameters involved in the error, and re-run a terraform plan. You sould get an output similar to this:

```bash

Terraform will perform the following actions:

  # azurerm_subnet.AzureBastionSubnet will be imported
    resource "azurerm_subnet" "AzureBastionSubnet" {
        address_prefixes                               = [
            "10.0.0.192/26",
        ]
        enforce_private_link_endpoint_network_policies = true
        enforce_private_link_service_network_policies  = false
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/AzureBastionSubnet"
        name                                           = "AzureBastionSubnet"
        private_endpoint_network_policies_enabled      = false
        private_link_service_network_policies_enabled  = true
        resource_group_name                            = "rsg_name"
        service_endpoint_policy_ids                    = []
        service_endpoints                              = []
        virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.subnet2 will be imported
    resource "azurerm_subnet" "subnet2" {
        address_prefixes                               = [
            "10.0.0.64/26",
        ]
        enforce_private_link_endpoint_network_policies = true
        enforce_private_link_service_network_policies  = false
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet2"
        name                                           = "subnet2"
        private_endpoint_network_policies_enabled      = false
        private_link_service_network_policies_enabled  = true
        resource_group_name                            = "rsg_name"
        service_endpoint_policy_ids                    = []
        service_endpoints                              = []
        virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.subnet3 will be imported
    resource "azurerm_subnet" "subnet3" {
        address_prefixes                               = [
            "10.0.0.128/26",
        ]
        enforce_private_link_endpoint_network_policies = true
        enforce_private_link_service_network_policies  = false
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet3"
        name                                           = "subnet3"
        private_endpoint_network_policies_enabled      = false
        private_link_service_network_policies_enabled  = true
        resource_group_name                            = "rsg_name"
        service_endpoint_policy_ids                    = []
        service_endpoints                              = []
        virtual_network_name                           = "vnet_name"
    }

Plan: 3 to import, 0 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if
you run "terraform apply" now.

```

Run the apply command to validate the import. After the apply is complete, check the state with a `terraform state list`.You should see the subnets in the state now:

```bash

df@df2204lts:~/Documents/myrepo/tf-intermediaire-lab2/src$ terraform state list
data.azurerm_resource_group.feature_rg
data.azurerm_resource_group.main
azurerm_subnet.AzureBastionSubnet
azurerm_subnet.subnet1
azurerm_subnet.subnet2
azurerm_subnet.subnet3
azurerm_virtual_network.main_vnet

```


### Exercise 5: Refactor the subnets terraform configuration with a for_each, and move the resources with a moved block

After the previous exercice, we have now, available in our terraform configuration, the 4 subnets defined as independant resources.
However, this is not optimized, and we would like to DRY this with a for_each.

To get started, we will comment all the import blocks in the `imports.tf` file.
To get a better view, we will also move all of our subnets configuration in a new file called `subnets.tf`

In this file, we will add the following configuration for the subnets:

```go

resource "azurerm_subnet" "vnet_name_subnets" {

  for_each = var.vnet_name_subnets_list

  name = each.key
  resource_group_name = data.azurerm_resource_group.self.name
  virtual_network_name = azurerm_virtual_network.main_vnet.name
  address_prefixes = [each.value.address_space]
}

```

We will also add the following variable in the `variable.tf` file:

```go

variable "vnet_name_subnets_list" {
  type = map(object({
    address_space = string
  }))
  description = "List of subnets create"

  default = {
    subnet1 = {
        address_space = "10.0.0.0/26"
    },
        subnet2 = {
        address_space = "10.0.0.64/26"
    },
        subnet3 = {
        address_space = "10.0.0.128/26"
    },
        AzureBastionSubnet = {
        address_space = "10.0.0.192/26"
    }
  }
}

```

Launching a terraform plan, we get something similar to this:

```bash

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_subnet.vnet_name_subnets["AzureBastionSubnet"] will be created
  + resource "azurerm_subnet" "vnet_name_subnets" {
      + address_prefixes                               = [
          + "10.0.0.192/26",
        ]
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "AzureBastionSubnet"
      + private_endpoint_network_policies_enabled      = (known after apply)
      + private_link_service_network_policies_enabled  = (known after apply)
      + resource_group_name                            = "rsg_name"
      + virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.vnet_name_subnets["subnet1"] will be created
  + resource "azurerm_subnet" "vnet_name_subnets" {
      + address_prefixes                               = [
          + "10.0.0.0/26",
        ]
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "subnet1"
      + private_endpoint_network_policies_enabled      = (known after apply)
      + private_link_service_network_policies_enabled  = (known after apply)
      + resource_group_name                            = "rsg_name"
      + virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.vnet_name_subnets["subnet2"] will be created
  + resource "azurerm_subnet" "vnet_name_subnets" {
      + address_prefixes                               = [
          + "10.0.0.64/26",
        ]
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "subnet2"
      + private_endpoint_network_policies_enabled      = (known after apply)
      + private_link_service_network_policies_enabled  = (known after apply)
      + resource_group_name                            = "rsg_name"
      + virtual_network_name                           = "vnet_name"
    }

  # azurerm_subnet.vnet_name_subnets["subnet3"] will be created
  + resource "azurerm_subnet" "vnet_name_subnets" {
      + address_prefixes                               = [
          + "10.0.0.128/26",
        ]
      + enforce_private_link_endpoint_network_policies = (known after apply)
      + enforce_private_link_service_network_policies  = (known after apply)
      + id                                             = (known after apply)
      + name                                           = "subnet3"
      + private_endpoint_network_policies_enabled      = (known after apply)
      + private_link_service_network_policies_enabled  = (known after apply)
      + resource_group_name                            = "rsg_name"
      + virtual_network_name                           = "vnet_name"
    }

Plan: 4 to add, 0 to change, 0 to destroy

```

where the subnets to be created already exist in our `subnets.tf` file, and in Azure.
The idea here is to somehow change the current configuration with the new one, without recreating the resources.
To achieve this, terraform state mv command can be used in an imperative way, or the moved block can be used in a declarative way.

The moved block is defined as below:

```go

moved {
  from = azurerm_<resource_type>.a
  to   = azurerm_<resource_type>.b
}

```

Ib the present case, we are considering moving the subnets in a terraform declaration with a `for_each` meta argument. We will add the following moved block:

```go

moved {
  from = azurerm_subnet.subnet1
  to = azurerm_subnet.hub_subnets["subnet1"]
}

moved {
  from = azurerm_subnet.subnet2
  to = azurerm_subnet.hub_subnets["subnet2"]
}

moved {
  from = azurerm_subnet.subnet3
  to = azurerm_subnet.hub_subnets["subnet3"]
}

moved {
  from = azurerm_subnet.AzureBastionSubnet
  to = azurerm_subnet.hub_subnets["AzureBastionSubnet"]
}

```

We also need to comment the first definition of the subnets and keep only the new one. The `subnets.tf` file should look like this:

```go

resource "azurerm_subnet" "hub_subnets" {

  for_each = var.hub_subnets_list

  name = each.key
  resource_group_name = data.azurerm_resource_group.self.name
  virtual_network_name = azurerm_virtual_network.main_vnet.name
  address_prefixes = [each.value.address_space]
}

moved {

  
  from = azurerm_subnet.subnet1
  to = azurerm_subnet.hub_subnets["subnet1"]
}

moved {
  from = azurerm_subnet.subnet2
  to = azurerm_subnet.hub_subnets["subnet2"]
}

moved {
  from = azurerm_subnet.subnet3
  to = azurerm_subnet.hub_subnets["subnet3"]
}

moved {
  from = azurerm_subnet.AzureBastionSubnet
  to = azurerm_subnet.hub_subnets["AzureBastionSubnet"]
}

```

Run a terraform plan. The output should be similar to this:

```bash

Terraform will perform the following actions:

  # azurerm_subnet.AzureBastionSubnet has moved to azurerm_subnet.hub_subnets["AzureBastionSubnet"]
    resource "azurerm_subnet" "hub_subnets" {
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/AzureBastionSubnet"
        name                                           = "AzureBastionSubnet"
        # (9 unchanged attributes hidden)
    }

  # azurerm_subnet.subnet1 has moved to azurerm_subnet.hub_subnets["subnet1"]
    resource "azurerm_subnet" "hub_subnets" {
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet1"
        name                                           = "subnet1"
        # (9 unchanged attributes hidden)
    }

  # azurerm_subnet.subnet2 has moved to azurerm_subnet.hub_subnets["subnet2"]
    resource "azurerm_subnet" "hub_subnets" {
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet2"
        name                                           = "subnet2"
        # (9 unchanged attributes hidden)
    }

  # azurerm_subnet.subnet3 has moved to azurerm_subnet.hub_subnets["subnet3"]
    resource "azurerm_subnet" "hub_subnets" {
        id                                             = "/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/rsg_name/providers/Microsoft.Network/virtualNetworks/vnet_name/subnets/subnet3"
        name                                           = "subnet3"
        # (9 unchanged attributes hidden)
    }

Plan: 0 to add, 0 to change, 0 to destroy.

─────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if
you run "terraform apply" now.

```

Notice that no change is planned. Notice also the `azurerm_subnet.subnet_name has moved to azurerm_subnet.hub_subnets["subnet_name"]` which indicates the move to the new place in the state, without any rela change on the infrastructure.
Run a terraform apply to validate the change.

```bash

Plan: 0 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes


Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

```

check the state to validate the move:

```bash

yumemaru [ ~ ]$ terraform state list
data.azurerm_resource_group.feature_rg
data.azurerm_resource_group.main
azurerm_subnet.hub_subnets["AzureBastionSubnet"]
azurerm_subnet.hub_subnets["subnet1"]
azurerm_subnet.hub_subnets["subnet2"]
azurerm_subnet.hub_subnets["subnet3"]
azurerm_virtual_network.main_vnet


```

Remove the resource with a terraform destroy command.