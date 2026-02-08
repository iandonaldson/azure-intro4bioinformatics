# azure-intro4bioinformatics

notes from this path

### 1. Azure cloud concepts  
Read the learning path [here](https://learn.microsoft.com/en-us/training/paths/microsoft-azure-fundamentals-describe-cloud-concepts).  The azure_cloud_concepts.md file contains a glossary of basic cloud terms and concepts related to azure. 

### 2a. Linux on azure  
Read through the learning path [here](https://learn.microsoft.com/en-us/training/paths/azure-linux/).  The linux_on_azure.md file contains a glossary of terms and concepts specific to Linux deployment and administration through Azure.

### 2b. Practical Deployment of Azure VMs  

Deploying and managing Azure virtual machines (VMs) involves using the Azure Portal to create resources—specifically selecting an OS image, VM size, and networking—within a defined resource group. Key management tasks include connecting via RDP/SSH, monitoring performance, utilizing Azure Bastion for security, and implementing auto-shutdown to control costs. 

You can use the practical notes in 2.b.practical_linux_deployment_on_azure.md as a guide to setting up a Linux VM to carry out a typical STAR alignment on a single VM.  Other tasks will require modified VM's where CPU/RAM will be chosen based on the workload requirements.
  
| Tool        | Azure concept              |
| ----------- | -------------------------- |
| STAR        | High-memory VM + fast disk |
| Cell Ranger | Many cores + local NVMe    |
| Seurat      | Modest VM, RStudio Server  |
| Snakemake   | VM + local execution       |
| Nextflow    | VM + Docker/Singularity    |
  

### 3a. Storage on Azure
Read through this background material on setting up storage on Azure [here](https://learn.microsoft.com/en-us/training/paths/store-data-in-azure/).  The notes at storage_on_azure.md provide a summary of terms and concepts with a bioinformatics focus.

### 3b. Practical Deployment of storage on Azure 

You can use the notes in 3.b.practical_storage_deployment_on_azure.md as a guide to setting up a storage account on azure.  (These notes are based on this [tutorial](https://learn.microsoft.com/en-us/training/modules/create-azure-storage-account/5-exercise-create-a-storage-account).  




