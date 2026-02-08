# azure-intro4bioinformatics

notes from this path

### 1. [azure cloud concepts](https://learn.microsoft.com/en-us/training/paths/microsoft-azure-fundamentals-describe-cloud-concepts)

### 2. [linux on azure](https://learn.microsoft.com/en-us/training/paths/azure-linux/)

### 3. Deploy Azure VMs

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
4. Set Credentials: Provide a username and password/SSH key for administrator access.  
5. Networking & Disks: Configure virtual networks, public IPs, and disk types (Standard SSD, Premium SSD).  
6. Review \+ Create: Validate the settings and click Create. 

Reference:  
[QuickLearn](https://portal.azure.com/#view/Microsoft_Azure_ComputeHub/QuickLearn.ReactView)

### 3. [storage on azure](https://learn.microsoft.com/en-us/training/paths/store-data-in-azure/)

