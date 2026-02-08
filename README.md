# azure-intro4bioinformatics

notes from this path

### 1. Azure cloud concepts  
Read the learning path [here](https://learn.microsoft.com/en-us/training/paths/microsoft-azure-fundamentals-describe-cloud-concepts).  The azure_cloud_concepts.md file contains a glossary of basic cloud terms and concepts related to azure. 

### 2a. Linux on azure  
Read through the learning path [here](https://learn.microsoft.com/en-us/training/paths/azure-linux/).  The linux_on_azure.md file contains a glossary of terms and concepts specific to Linux deployment and administration through Azure.

### 2b. Practical Deployment of Azure VMs  

Deploying and managing Azure virtual machines (VMs) involves using the Azure Portal to create resources—specifically selecting an OS image, VM size, and networking—within a defined resource group. Key management tasks include connecting via RDP/SSH, monitoring performance, utilizing Azure Bastion for security, and implementing auto-shutdown to control costs. 

How to Deploy Azure VMs

1. Sign in to the Azure Portal: Navigate to portal.azure.com.  
2. Initiate Creation: Search for "Virtual machines," click Create, and select Azure virtual machine.  
   [See this link](https://portal.azure.com/#view/Microsoft_Azure_ComputeHub/ComputeHubMenuBlade/~/virtualMachinesBrowse)   
3. Configure Basics:  
   1. Subscription/Resource Group: Select your subscription and create a new resource group for logical organization.  
   2. VM Name/Region: Name the VM and choose a geographic region.  
   3. Image: Select an OS image (e.g., Windows Server 2022, Ubuntu).  
   4. Size: Choose a VM size (CPU/RAM) based on workload requirements.
  
| Tool        | Azure concept              |
| ----------- | -------------------------- |
| STAR        | High-memory VM + fast disk |
| Cell Ranger | Many cores + local NVMe    |
| Seurat      | Modest VM, RStudio Server  |
| Snakemake   | VM + local execution       |
| Nextflow    | VM + Docker/Singularity    |
  
4. Set Credentials: Provide a username and password/SSH key for administrator access.  
5. Networking & Disks: Configure virtual networks, public IPs, and disk types (Standard SSD, Premium SSD).  
6. Review \+ Create: Validate the settings and click Create. 

Reference:  
[QuickLearn](https://portal.azure.com/#view/Microsoft_Azure_ComputeHub/QuickLearn.ReactView)

### 3a. Storage on Azure
Read through this background material on setting up storage on Azure [here](https://learn.microsoft.com/en-us/training/paths/store-data-in-azure/).  The notes at storage_on_azure.md provide a summary of terms and concepts with a bioinformatics focus.

### 3b. Practical Deployment of storage on Azure 

You can use the notes in 3.b.practical_storage_deployment_on_azure.md as a guide to setting up a storage account on azure.  (These notes are based on this [tutorial](https://learn.microsoft.com/en-us/training/modules/create-azure-storage-account/5-exercise-create-a-storage-account).  




