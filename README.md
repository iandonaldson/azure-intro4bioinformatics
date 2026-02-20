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


### 4. Administer Docker Containers on Azure

### 4a. Introduction to Docker Containers  
Read though this background material on docker containers on Azure [here](https://learn.microsoft.com/en-us/training/modules/intro-to-docker-containers/).  The notes at 4.a.docker_containers.md provide a summary of terms and concepts with a bioinformatics focus.  

### 4b. Practical Deployment of a website on Azure Using a Container.
You can use the notes in 4.b.practical_container_deployment_on_azure.md as a guide to setting up a container account on azure.  (These notes are based on this [tutorial]([https://learn.microsoft.com/en-us/training/modules/create-azure-storage-account/5-exercise-create-a-storage-account](https://learn.microsoft.com/en-us/training/modules/intro-to-containers/)). 

### 4c. Building and storing containers on Azure
Read though this background material on building and storing containers on Azure [here](https://learn.microsoft.com/en-us/training/modules/build-and-store-container-images/). The notes at 4.c.build_and_store_containers.md provide a summary of terms and concepts with a bioinformatics focus.  

### 4d. Basic bioinformatics image
For demonstration purposes only.  Consider https://hub.docker.com/r/broadinstitute/gatk 

### 4e. Modified basic image for STAR RNASeq
For demonstration purposes only.  Consider https://hub.docker.com/r/broadinstitute/gatk

### 4f. Introduction to Kubernetes
Read though this background material on Kubernetes [here](https://learn.microsoft.com/en-us/training/modules/intro-to-kubernetes/).
The notes at 4.d.kubernetes_intro.md provide a summary of terms and concepts with a bioinformatics focus.  

### 4g. Introduction to Azure Kubernetes Service (AKS)
Read though this background material on Azure Kubernetes Service [here](https://learn.microsoft.com/en-us/training/modules/intro-to-azure-kubernetes-service/).
The notes at 4.d.azure_kubernetes_service.md provide a summary of terms and concepts with a bioinformatics focus.  


### High-performance computing on Azure  
[here](https://learn.microsoft.com/en-us/azure/architecture/topics/high-performance-computing)  
[tutorial](https://learn.microsoft.com/en-us/training/paths/run-high-performance-computing-applications-azure)  

### Nextflow and Azure Batch  
[here](https://learn.microsoft.com/en-us/azure/batch/)  
[here](https://www.nextflow.io/docs/latest/azure.html)  

### Snakemake and Azure Batch  
[here](https://learn.microsoft.com/en-us/azure/batch/)  
[tutorial](https://snakemake.readthedocs.io/en/v7.30.1/executor_tutorial/azure_batch.html)  
[snakemake-plugin](https://snakemake.github.io/snakemake-plugin-catalog/plugins/executor/azure-batch.html)   



### Optional  
[Managing Azure](https://learn.microsoft.com/en-us/training/paths/az-104-manage-compute-resources/)  
[Azure Architecture Patterns](https://learn.microsoft.com/en-us/azure/architecture)  

### Other learning paths  
[Data Scientist](https://learn.microsoft.com/en-us/training/career-paths/data-scientist)  
[AI Engineer](https://learn.microsoft.com/en-us/training/career-paths/ai-engineer)  
[Data Engineer](https://learn.microsoft.com/en-us/training/career-paths/data-engineer)  
[DevOps Engineer](https://learn.microsoft.com/en-us/training/career-paths/devops-engineer)
[Solutions Architect](https://learn.microsoft.com/en-us/training/career-paths/solution-architect)



[Progress](https://learn.microsoft.com/en-us/users/iandonaldson-2790/)


