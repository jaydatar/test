
# Golden Images with Packer and Azure DevOps Pipeline

## Introduction
Golden images are pre-configured virtual machine images that include necessary software, configurations, and security settings. Automating their creation using Packer and Azure DevOps (ADO) pipelines ensures consistency, reduces deployment time, and streamlines maintenance. In the future, integration with Ansible can further enhance configuration management.

This document outlines how to create and manage golden images using Packer and automate the process through an ADO pipeline.

## Scope
This document focuses on building Linux-based golden images using Packer and automating the image build process using an ADO pipeline. The scope includes setting up the Packer template, configuring the ADO pipeline, and storing images in an Azure Compute Gallery (formerly known as Shared Image Gallery). The integration with Ansible for post-build configuration is out of scope but will be part of future enhancements.

## Packer Configuration

### Prerequisites
- Packer installed on a local machine or ADO-hosted agent.
- Access to an Azure account with appropriate permissions (Contributor or Owner).
- A resource group and storage account for managing Packer's state.

### Packer Template Setup
Create a Packer template (`template.pkr.hcl`) to define the base image, provisioning steps, and image configuration:

```hcl
# Packer configuration for building a golden image in Azure

source "azure-arm" "example" {
  azure_tags           = {
    "Environment" = "Production"
  }
  managed_image_name   = "golden-image"
  managed_image_rg     = "packer-resource-group"
  location             = "East US"
  vm_size              = "Standard_D2s_v3"

  os_type              = "Linux"
  image_publisher      = "Canonical"
  image_offer          = "UbuntuServer"
  image_sku            = "18.04-LTS"
  # Credentials for Azure
  client_id            = var.client_id
  client_secret        = var.client_secret
  subscription_id      = var.subscription_id
  tenant_id            = var.tenant_id
}

build {
  sources = ["source.azure-arm.example"]

  provisioner "shell" {
    inline = [
      "sudo apt-get update -y",
      "sudo apt-get install -y nginx"
    ]
  }
}
```

### Packer Build Command
You can test the Packer template locally by running the following command:

```bash
packer init .
packer build -var 'client_id=your_client_id' -var 'client_secret=your_secret' template.pkr.hcl
```

### Ansible Integration (Future Work)
In the future, Ansible will be used for more advanced provisioning tasks, such as installing complex software or applying specific configurations. This will allow post-image customization and further automation of infrastructure configuration.

## Azure DevOps Pipeline Setup

### Pipeline Configuration
To automate the creation of the golden image, configure an ADO pipeline with Packer using the following YAML pipeline file (`azure-pipelines.yml`):

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  ARM_CLIENT_ID: $(client_id)
  ARM_CLIENT_SECRET: $(client_secret)
  ARM_SUBSCRIPTION_ID: $(subscription_id)
  ARM_TENANT_ID: $(tenant_id)

steps:
- task: PackerInstaller@1
  displayName: 'Install Packer'

- script: |
    packer init .
    packer build -var "client_id=$(ARM_CLIENT_ID)" -var "client_secret=$(ARM_CLIENT_SECRET)" template.pkr.hcl
  displayName: 'Build Image with Packer'
```

### Variables
Ensure the following variables are defined in the ADO pipeline:
- `ARM_CLIENT_ID`: Azure service principal client ID.
- `ARM_CLIENT_SECRET`: Service principal secret.
- `ARM_SUBSCRIPTION_ID`: Subscription ID.
- `ARM_TENANT_ID`: Tenant ID for the Azure account.

## Authentication with Azure Service Principal

Packer requires access to your Azure account to create and manage resources such as virtual machines and images. To achieve this, Packer can authenticate using an Azure service principal. A service principal is a security identity used by applications or services to access specific Azure resources.

#### Using Azure Service Principal with Packer

To authenticate with Azure, Packer requires the following details from the service principal:

- **Client ID**: The unique identifier for the application (service principal).
- **Client Secret**: The password associated with the service principal.
- **Subscription ID**: The Azure subscription ID where the resources will be managed.
- **Tenant ID**: The ID of the Azure Active Directory tenant.

You can create a service principal using the Azure CLI:

```bash
az ad sp create-for-rbac --name "PackerSP" --role Contributor --scopes /subscriptions/<subscription-id>
```

This command will output the `clientId`, `clientSecret`, `tenantId`, and `subscriptionId`, which are used in Packer's configuration.

The Packer template would reference these variables as shown:

```hcl
source "azure-arm" "example" {
  client_id            = var.client_id
  client_secret        = var.client_secret
  subscription_id      = var.subscription_id
  tenant_id            = var.tenant_id
  ...
}
```

Make sure to securely store and manage the service principal credentials. In ADO pipelines, these credentials are stored as environment variables or secret variables to avoid exposing them in plaintext.

#### Example Configuration in ADO Pipeline:

```yaml
variables:
  ARM_CLIENT_ID: $(client_id)
  ARM_CLIENT_SECRET: $(client_secret)
  ARM_SUBSCRIPTION_ID: $(subscription_id)
  ARM_TENANT_ID: $(tenant_id)

steps:
- task: PackerInstaller@1
  displayName: 'Install Packer'

- script: |
    packer init .
    packer build -var "client_id=$(ARM_CLIENT_ID)" -var "client_secret=$(ARM_CLIENT_SECRET)" template.pkr.hcl
  displayName: 'Build Image with Packer'
```

### Federated Identity vs Service Principal Authentication

#### Federated Authentication and ADO

Federated authentication allows Azure DevOps (ADO) to interact with Azure resources without storing credentials directly. Instead, it uses a trust relationship between ADO and Azure Active Directory (AAD). This approach eliminates the need for storing secrets and credentials like `client_secret` and provides more secure, short-lived tokens for authentication.

However, **Packer does not yet support federated identity-based authentication** with Azure in the same way Azure Pipelines can use managed identities or federated tokens. The main challenge with federated identity for Packer is that Packer relies on long-lived credentials such as service principal secrets, while federated authentication relies on short-lived tokens that are automatically refreshed. Since Packer operates outside of ADO's identity framework, it cannot obtain or manage these tokens natively.

#### Why Federated Authentication Does Not Work with Packer in ADO

- **Token Expiry**: Federated identities use short-lived tokens that need frequent refreshing, whereas Packer expects static credentials like service principal secrets to remain valid for the duration of the build process.
  
- **No Managed Identity Integration**: Packer currently does not natively support managed identities or federated identity credentials. It expects the credentials (client secret) to be manually supplied and cannot leverage the federated identity workflow that ADO pipelines might use.

#### Solution: Using Service Principal with Secrets

For now, the most reliable method to authenticate Packer with Azure in an ADO pipeline is by using a service principal with a `client_secret`. This secret can be stored securely in Azure Key Vault or as a secret variable in ADO pipelines, ensuring it’s only accessible during the build process.

In future Packer updates, support for managed identities or federated authentication may be introduced, but until then, using service principals is the recommended approach for reliable and secure authentication.

## Performance and Maintenance
Using Packer ensures fast, consistent, and reliable image builds. By automating the image creation process with ADO pipelines, you can maintain and update golden images efficiently. In the future, integrating Ansible will allow further fine-tuning of configurations after the image has been built.

## Conclusion
Building golden images using Packer and Azure DevOps pipeline automation improves deployment consistency and speeds up the VM provisioning process. The future integration of Ansible will provide an additional layer of configuration management.

## Resources
- [Packer Documentation](https://www.packer.io/docs)
- [Azure DevOps Pipeline Documentation](https://learn.microsoft.com/en-us/azure/devops/pipelines/?view=azure-devops)
- [Azure Shared Image Gallery](https://learn.microsoft.com/en-us/azure/virtual-machines/shared-image-galleries)
- [Ansible Documentation](https://docs.ansible.com/)
