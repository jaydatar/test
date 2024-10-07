# Architecture Approach Document for Virtual Machine Golden Images

Version: 1.0.0

Date: 03/05/2024

Status: **Recommendation**

## Document Control

| Title                   | Document Author | Product Manager  |
|-------------------------|-----------------|------------------|
| VM Golden Images        | Neil Moorthy    | James Waterfield |

## Change History

| Version | Status                      | Author/Editor | Change (brief summary of change)                                |
|---------|-----------------------------|---------------|-----------------------------------------------------------------|
| 1.0.0   | **Proposed Recommendation** | Neil Moorthy   |                                                                 |

## Table of Contents

- [Architecture Approach Document for Virtual Machine Golden Images](#Architecture-Approach-Document-for-Virtual-Machine-Golden-Images)
- [Document Control](#document-control)
- [Change History](#change-history)
- [Table of Contents](#table-of-contents)
- [Project Approach](#project-approach)
  - [Current problem](#current-problem)
  - [Scope](#scope)
    - [Ansible tasks](#ansible-tasks)
  - [Options](#options)
    - [Options Summary](#options-summary)
    - [Option 1 - Ansible only-Build from built-in image each time](#option-1---ansible-only-build-from-built-in-image-each-time)Ansible only, 
      - [Summary](#summary)
      - [Value](#value)
      - [Cost](#cost)
    - [Option 2 - Both Azure VM Builder and Ansible](#option-2---both-azure-vm-builder-and-ansible)
      - [Summary](#summary-1)
      - [Value](#value-1)
      - [Cost](#cost-1)
    - [Option 3 - Both Hashicorp Packer and Ansible](#option-3---both-hashicorp-packer-and-ansible)
      - [Summary](#summary-2)
      - [Value](#value-2)
      - [Cost](#cost-2)
    - [Preferred Option](#preferred-option)
    - [Proof of Concept (PoC)](#proof-of-concept)


# Project Approach


## Current problem

Azure Virtual Machine servers (VMs) are currently deployed using built-in Azure marketplace VM images. Any subsequent changes are applied using Ansible, for example installing agents, security updates etc. 

In order to speed-up and streamline VM deployments, some Ansible tasks can potentially be pre-staged to a golden image. The golden image would include the more repetitive and timely tasks that are applied to all VMs. This approach, known as Images as Code can be combined with Configuration Management to deploy a VM.

Currently, time-consuming tasks such as downloading updates and installers, hardening virtual machines etc, are peformed via Ansible tasks each time a virtual machine is deployed. These tasks can be performed once and built into a re-usable golden image. This reduces deployment time, risk of transient deploy-time failures etc.

The Azure compute gallery provides an image storage facility where golden images can be deployed from. It also supports versioning, global replication, cross tenant sharing, multi-region and subscription deployments, release notes, Role Based Access Control (Rbac) permission model etc.

The combination of tools including: version control (Azure DevOps, GitHub, GitLab etc), an image creation tool, configuration management (Ansible) and the Azure compute gallery will provide a more capable VM deployment pipeline.

Further reading:

- [Azure Compute Gallery](https://learn.microsoft.com/en-us/azure/virtual-machines/azure-compute-gallery)


## Scope

Azure Server VM's operating system vendors include Microsoft, Red Hat, Canonical etc. Client operating systems such as Microsoft Windows 10 / 11, Mac OS etc. are out of scope.


### Ansible tasks

The following Anisble tasks are applied to all VM instances. While we do not store long term Ansible task logs and timings, the following task list / timing estimates have been gathered by manually checking some recent pipeline runs...


- Server Configuration
  - Linux (takes around 11 minutes to complete):
    - Setup local admins
    - Set timezone
    - Set locale settings (key board layout etc)
    - Install Defender Endpoint agent
    - Setup anti-virus agent configuration
    - Run apt update (update installed software)
    - Set NTP servers
    - Setup DNS / DHCP
    - Domain join VM
    - Install observability agents (elastic agent)
    - Install commvault backup agent
  - Windows:
    - Set timezone
    - Set disk / drive letter / OS label
    - Set bginfo (desktop background - sysinfo)
    - Install Powershell
    - Disable Customer Experience Improvement Program (CEIP)
    - Disable SMB1 protocol
    - Install Defender Endpoint agent
    - Setup DNS / DHCP
    - Set locale settings
    - Setup Windows Firewall
    - Domain join VM
    - Set local administrators
    - Install observability agents (elastic agent)
    - Install commvault backup agent
    - Install Windows updates


- Server Hardening roles (note that the latest CIS version - 2.2.0, will be used in the near future)
  - ao.server_hardening.UBUNTU20_CIS_1_1_0 (takes around 47 minutes to complete):
    - (this also includes: RHEL8_CIS_2_0_0)
    - Secure system accounts / groups (takes around 20 minutes)
    - Install AIDE (Advanced Intrusion Detection Environment)
    - Secure user accounts
    - Remove non-essential services
    - Secure network and firewall
    - Setup auditing
    - Setup sudo / SSH / PAM
    - Set file / user / group permissions
  - ao.server_hardening.Windows_2019_CIS_1_3_0:
    - (this also includes: Windows_2016_CIS_1_2_0)
    - Secure system accounts / groups
    - Setup account permissions
    - Secure network and firewall
    - Setup auditing
    - Disable smb1 and other non-essential protocol / processes
    - Install latest updates
  - ao.server_hardning.sql2019
    - Setup SQL admin user
    - Install .Net framework
    - Create SQL directories
    - Download/mount Windows iso file
    - Run enable SQL TCP script
    - Restart SQL instance


Further reading:

- [CIS image benchmarks](https://www.cisecurity.org/benchmark)


## Options

### Options Summary

- ***Option 1***; Ansible only-build from built-in image each time
- ***Option 2***; Both Azure VM Builder and Ansible
- ***Option 3***; Both Hashicorp Packer and Ansible

### Option 1 - Ansible only, build from built-in image each time

#### Summary

This option is the current as-is solution used to deploy virtual machines. It has the downside of repeating the same timely tasks each time we deploy a VM. We carry out the above Ansible tasks everytime we deploy a virtual machine. As a result deployment times are high (circa 1 hour per VM - server configuration ~ 11 mins + server hardening ~ 47 mins). 

As an example, when installing operating system updates and security patches we download and install them each time we deploy a VM. 

Ansible provides flexibilty at deployment-time, as well as post-deployment, for example deploying certificates or keys etc. Ansible is ideal for configuration which is unique per VM deployment. However, for tasks which are identical across all VM deployments, such as hardening, installing updates or downloading / installing agents, repeating these tasks per pipeline run is expensive and takes time.

For server hardening CIS provide a limited subset of pre-built VM images available on the Azure marketplace at cost. These could be utilised in some short-lived cases to speed up current deployment times.


#### Value

***For***:

- Flexible and ideal for deploying configurations unique to a specific VM deployment.
- Can run deployments / configurations against running servers.
- Can be run at any time (post-build) against a VM to check for drift.

***Against***:

- Expensive and time-consuming for tasks which are consistent across all VM deployments and can be done up-front.


#### Cost

- Compute costs associated with long pipeline run-times.
- Time / resource cost of waiting for long VM deployment times.



### Option 2 - Both Azure VM Builder and Ansible

#### Summary

This option combines using an Azure VM Builder dervied image with Ansible. Azure VM Builder is built on-top of Hashicorp Packer. It is a relatively new tool and is Azure specific and hence relatively niche. 

Azure VM Builder is a hosted Azure service accessed via an API. Configuration is defined in Json / Bicep. Images can be stored in the Azure Compute Gallery. Azure VM builder provides pipeline tasks for CI / CD.

Azure VM Builder is not open source and hence has a much smaller community / support. It also lacks support for building images as code (IaC) / configuration as code (CaC) on other platforms, i.e. VMWare etc.

Azure VM Builders provisioner only supports Powershell.

Further reading:

- [Azure Image Builder](https://learn.microsoft.com/en-us/azure/virtual-machines/image-builder-overview?tabs=azure-powershell)


#### Value

***For***:

- Platform as a Service (PaaS) so easy to setup.

***Against***:

- Cannot be used outside of Azure.
- Lacks integrations with Ansible etc.
- Less mature, small community.
- Uses Json / Bicep for configuration.
- Limited provisioner language support.
- Supporting, updating and distributing golden images.


#### Cost

- Azure VM Builder is a free service.
- Azure compute gallery storage / egress costs.



### Option 3 - Both Hashicorp Packer and Ansible

#### Summary

This option combines using a Hasicorp Packer dervied image with Ansible. Packer is a widely used mature product that uses Hashicorp Configuration Language (HCL) to define image configurations for multiple platforms. We can use it to build Azure VM images in parallel and deploy to the Azure Compute Gallery. It is widely supported for CI / CD pipelines.

Packer includes an Ansible Packer provisioner which can be used to run Ansible playbooks. This allows us to call an Ansible Playbook after our image is created. 

The Packer provisioner supports Powershell, Ansible, Chef, Puppet, DSC etc.

Packer also includes an Azure integration and Azure resource manager builder which can authenticate using a Service Principal, or virtual machine identity. We can also define not just the image definition but also reference the Azure compute gallery using the integration.

Further reading:

- [Packer - Azure integration](https://developer.hashicorp.com/packer/integrations/hashicorp/azure)
- [Packer - Ansible integration](https://developer.hashicorp.com/packer/integrations/hashicorp/ansible/latest/components/provisioner/ansible)
- [Packer Azure builder](https://developer.hashicorp.com/packer/integrations/hashicorp/azure/latest/components/builder/arm)
- [Packer HCL Guide](https://developer.hashicorp.com/packer/guides/hcl)



#### Value

***For***:

- Uses HCL language for image configuration which provides a more flexible and concise syntax than JSON.
- Supports multiple platforms via plugins, including building Docker images.
- Mature and widely used / supported.
- Open source.
- Includes integration with Ansible provisioner.
- Can be used to deploy complex workload specific images.

***Against***:

- Requires supporting / maintaining own infrastructure (ADO agents)
- Supporting, updating and distributing golden images.


#### Cost

- Packer is open source / free.
- Azure compute gallery storage / egress costs.
  - Each image is charged as a managed disk snapshot. Based on the existing Ansible VM roles: Windows Server 2016, Windows Server 2019, Windows Server 2022, Red Hat EL 8, Ubuntu 18.04, Ubuntu 20.04, Elastic Beats, ELK Stack, SQL Server 2019, we have 9 x 127GB OS images, if each consumes ~30GB space and we replicate to two secondary regions (i.e. US and Asia), we have 27 x 30GB = 810GB storage. A 512GB managed disk per region (x3) for snapshot storage costs ~£213.30 (£71.10 x 3) a month.


Further reading:

- [Azure Compute Gallery Costs](https://learn.microsoft.com/en-us/azure/virtual-machines/azure-compute-gallery#billing)



### Preferred Option

The prefered option is Packer for creation of a set of golden images which can then be consumed from the Azure Compute Gallery. Packer provides a mature, cross platform, image building tool with strong standards and support via an open source community.

Packer can be called from Azure DevOps pipelines and includes integrations with Ansible. The combination of Packer, Azure DevOps, Ansible and the Azure Compute Gallery allows us to build up a library of pre-built custom VM images which can be easily consumed.

The Azure Compute Gallery supports Rbac for access control. While Rbac permissions can be assigned for individual image versions it is recommended to control access at the Gallery scope (read, write etc).

A proof of concept (PoC) should be conducted to help determine configuration management tasks that could be pre-staged with Packer in a golden image. The PoC would also include distribution and deployment of images and integration of Packer with Ansible / Azure.

![Azure Packer CICD](./assets/img/virtual-machine-azure-package-ansible.png)

Further reading:

[Azure Compute Gallery - Rbac](https://learn.microsoft.com/en-us/azure/virtual-machines/share-gallery?tabs=portal#share-using-rbac)


### Proof of Concept

A Proof of Concept (PoC) should be conducted, to among other things, confirm which parts of a VM build should be performed by which component. Candidate parts of a PoC include:

- Use Packer to build a simple Linux image
- Deploy above Packer image to the Azure Compute Gallery
- Create an end-to-end Azure DevOps pipeline; that uses Packer to create and deploy VM image to the Azure Compute Gallery, then deploy the image via existing as-is Bicep template and run existing as-is Ansible run-time configuration on-top.
- Create a hardended Windows/Linux image using Packer ,deploy and test.
- Create market-place based image using Packer (i.e. SQL Server)
- Depending on time, test a subset of current as-is Ansible tasks migrated to Packer.