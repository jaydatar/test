
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

## Performance and Maintenance
Using Packer ensures fast, consistent, and reliable image builds. By automating the image creation process with ADO pipelines, you can maintain and update golden images efficiently. In the future, integrating Ansible will allow further fine-tuning of configurations after the image has been built.

## Conclusion
Building golden images using Packer and Azure DevOps pipeline automation improves deployment consistency and speeds up the VM provisioning process. The future integration of Ansible will provide an additional layer of configuration management.

## Resources
- [Packer Documentation](https://www.packer.io/docs)
- [Azure DevOps Pipeline Documentation](https://learn.microsoft.com/en-us/azure/devops/pipelines/?view=azure-devops)
- [Azure Shared Image Gallery](https://learn.microsoft.com/en-us/azure/virtual-machines/shared-image-galleries)
- [Ansible Documentation](https://docs.ansible.com/)
