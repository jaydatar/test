
# Golden Images with Packer and Azure DevOps Pipeline

Version 0.0.1

Date 07/10/24

| Title                     | Document Author    | Product Manager  | Iteration                                                                                                      |
| ------------------------- | ------------------ | ---------------- | -------------------------------------------------------------------------------------------------------------- |
| Golden Images with Packer and Azure DevOps Pipeline | Tapan Patel| James Waterfield | [Azure 2.2](https://allenovery.visualstudio.com/Platform%20Team/_wiki/wikis/Platform-Team.wiki/7280/Azure-2.2) |

## Introduction
Golden images are pre-configured virtual machine images that include necessary software, configurations, and security settings. Automating their creation using Packer and Azure DevOps (ADO) pipelines ensures consistency, reduces deployment time, and streamlines maintenance. In the future, integration with Ansible can further enhance configuration management.

This document outlines how to create and manage golden images using Packer and automate the process through an ADO pipeline.

## Scope
This document focuses on building Linux-based golden images using Packer and automating the image build process using an ADO pipeline. The scope includes setting up the Packer template, configuring the ADO pipeline, and storing images in an Azure Compute Gallery (formerly known as Shared Image Gallery). The integration with Ansible for post-build configuration is out of scope but will be part of future enhancements.

## Packer Configuration

### Prerequisites
- Packer installed on a local machine or ADO-hosted agent.
- Access to an Azure account with appropriate permissions (Contributor or Owner).
- A resource group and Azure Compute Gallery Instance for managing Packer images.

### Packer Template Setup
Create a Packer template (`ubuntu.pkr.hcl`) to define the base image, provisioning steps, and image configuration:

```hcl
# Packer configuration for building a golden image in Azure

source "azure-arm" "ubuntu_20" {
  client_id        = var.client_id
  client_secret    = var.client_secret
  tenant_id        = var.tenant_id
  subscription_id  = var.subscription_id

  managed_image_resource_group_name = var.resource_group
  managed_image_name                = "ubuntu-20-image"
  location                          = var.location

  shared_image_gallery_destination {
  subscription        = var.subscription_id
  resource_group      = var.resource_group
  gallery_name        = var.compute_gallery_name
  image_name          = var.compute_gallery_image_name
  image_version       = var.compute_gallery_image_version
  }

  vm_size         = "Standard_D2s_v3"
  communicator    = "ssh"
  os_type         = "Linux"
  image_publisher = "Canonical"
  image_offer     = "0001-com-ubuntu-server-focal"
  image_sku       = "20_04-lts-gen2"
  image_version   = "latest"

  azure_tags = {
    Created-by = "Packer"
    OS_Version = "Ubuntu 20.04"
    Release    = "Latest"
  }
}

build {
  sources = ["source.azure-arm.ubuntu_20"]

  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get upgrade -y",
      "apt-get install ansible -y",
      "sudo apt-get install -y python3-pip ansible",  # Install Ansible on the instance
      "sudo ansible-galaxy collection install community.general",  # Install collection on the instance
    ]
  }
}

```

### Packer Build Command
You can test the Packer template locally by running the following command:

```bash
packer init .
packer build -var 'client_id=your_client_id' -var 'client_secret=your_secret' ubuntu_20.pkr.hcl
```

### Ansible Integration (Future Work)
In the future, Ansible will be used for more advanced provisioning tasks, such as installing complex software or applying specific configurations. This will allow post-image customization and further automation of infrastructure configuration.

## Azure DevOps Pipeline Setup

### Pipeline Configuration
To automate the creation of the golden image, configure an ADO pipeline with Packer using the following YAML pipeline file (`packer_build.yaml`):

```yaml
name: $(BuildDefinitionName)_$(DayOfMonth).$(Month).$(Year:yyyy)$(Rev:.r)

pr: none

trigger: none

variables:
  - group: vg-packer-poc
  
pool:
  # name: $(pool_name)
  vmImage: ubuntu-latest
  
stages :
  - stage: packer_build
    jobs:
    - job: packer_build_job
      #continueOnError: false
      steps:
        - checkout: self
          fetchDepth: 25

        - task: PackerTool@0
          displayName: 'packer install'        
          inputs:
            version: '1.11.2'

        - task: Packer@1
          displayName: 'packer init'
          inputs:
            connectedServiceType: 'azure'
            azureSubscription: 'sp-tp-dev-01'
            templatePath: './packer/linux/ubuntu/20_04_LTS/'
            command: 'init'

        - task: Packer@1
          displayName: 'packer build'
          inputs:
            connectedServiceType: 'azure'
            azureSubscription: 'sp-tp-dev-01'
            templatePath: './packer/linux/ubuntu/20_04_LTS/'
            command: 'build'
            # variables:
            #   var = '1'
```

### Variables
Ensure the following variables are passed from the ADO service connection to the packer task in the ADO pipeline:
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
