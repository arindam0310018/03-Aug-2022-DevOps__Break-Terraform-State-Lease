# BREAK TERRAFORM STATE LEASE USING AZURE DEVOPS

Greetings to my fellow Technology Advocates and Specialists.

In this Session, I will demonstrate how to Break Terraform State Lease Using Azure DevOps.

| __USE CASE:-__ |
| --------- |
| In Order to Protect State File from Accidental Deletion or Tampering, Direct User Access to Terraform State File is Prohibited. |
| While Build IaC [Infrastructure-As-Code] Using Terraform, DevOps Engineer tend to Run the Code locally by manually executing Terraform Init, Plan and Apply Commands respectively. |
| During this whole Process, there might be Situation, where the Terraform State file is in Locked State and Unless the Lock is released, the code cannot be executed anymore (Manually or using Az DevOps Pipeline). |
| This is where, the below Az DevOps Pipeline helps. |
| The Az DevOps Pipeline runs in the Build Agent using Az DevOps Service Connection which is Az Service Principal Credentials behind the Scene with Appropriate RBAC [Role Based Access Control] on Subscription or Resource Group Level. |

| __AUTOMATION OBJECTIVE:-__ |
| --------- |
| Validate If Resource Group Exists. |
| Validate If Storage Account Exists. |
| Validate If Storage Account Container Exists. |
| Validate If Terraform State File Exists in the Specified Storage Account Container. |
| If any One of the above validation __DOES NOT PASS__, Pipeline will Fail immediately. |
| If All of the above validation is __SUCCESSFUL__, Pipeline will then check the Terraform Blob State. |
| If Terraform Blob State is == LEASED, Pipeline will Break the Lease. |
| If Terraform Blob State is != LEASED, Pipeline Still executes Successfully without altering the present state. |

| __IMPORTANT TO NOTE:-__ |
| --------- |
| There is No way to find the Blob Lease State before executing the __az storage blob lease break__ command. |
| Refer the Link for more Information: [az cli blob lease break](https://docs.microsoft.com/en-us/cli/azure/storage/blob/lease?view=azure-cli-latest#az-storage-blob-lease-break) |

| __REQUIREMENTS:-__ |
| --------- |

1. Azure Subscription.
2. Azure DevOps Organisation and Project.
3. Service Principal with Required RBAC ( __Contributor__) applied on Subscription or Resource Group(s).
4. Azure Resource Manager Service Connection in Azure DevOps.
5. Microsoft DevLabs Terraform Extension Installed in Azure DevOps and in Local System (VS Code Extension).


| __HOW DOES MY CODE PLACEHOLDER LOOKS LIKE:-__ |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/v2scnn4sl9er683c0h6s.png) |

| PIPELINE CODE SNIPPET:- | 
| --------- |

| AZURE DEVOPS YAML PIPELINE (azure-pipelines-tf-break-lease-v1.0.yml):- | 
| --------- |

```
trigger:
  none

######################
#DECLARE PARAMETERS:-
######################
parameters:
- name: SubscriptionID
  displayName: Subscription ID Details Follow Below:-
  type: string
  default: 210e66cb-55cf-424e-8daa-6cad804ab604
  values:
  - 210e66cb-55cf-424e-8daa-6cad804ab604

- name: RGNAME
  displayName: Please Provide the Resource Group Name:-
  type: object
  default: 

- name: STORAGEACCOUNTNAME
  displayName: Please Provide the Storage Account Name:-
  type: object
  default: 

- name: STORAGEACCOUNTCONTAINERNAME
  displayName: Please Provide the Storage Account Container Name:-
  type: object
  default:

- name: TFSTATEFILENAME
  displayName: Please Provide the TF State Name (For Example - "abc.tfstate" OR "Folder-A\abc.tfstate"):-
  type: object
  default: 

######################
#DECLARE VARIABLES:-
######################
variables:
  ServiceConnection: amcloud-cicd-service-connection
  BuildAgent: windows-latest

#########################
# Declare Build Agents:-
#########################
pool:
  vmImage: $(BuildAgent)

###################
# Declare Stages:-
###################

stages:

- stage: BREAK_TF_LEASE 
  jobs:
  - job: BREAK_TF_LEASE 
    displayName: BREAK TERRAFORM LEASE
    steps:
    - task: AzureCLI@2
      displayName: VALIDATE AND BREAK LEASE
      inputs:
        azureSubscription: $(ServiceConnection)
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          az --version
          az account set --subscription ${{ parameters.SubscriptionID }}
          az account show  
          $i = az group exists -n ${{ parameters.RGNAME }}
            if ($i -eq "true") {
              echo "#####################################################"
              echo "Resource Group ${{ parameters.RGNAME }} exists!!!"
              echo "#####################################################"
              $j = az storage account check-name --name ${{ parameters.STORAGEACCOUNTNAME }} --query "reason" --out tsv
                if ($j -eq "AlreadyExists") {
                  echo "#####################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} exists!!!"
                  echo "#####################################################"  
                  $k = az storage container exists --account-name ${{ parameters.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --name ${{ parameters.STORAGEACCOUNTCONTAINERNAME }} --query "exists" --out tsv 
                    if ($k -eq "true") {
                      echo "#####################################################"
                      echo "Storage Account Container ${{ parameters.STORAGEACCOUNTCONTAINERNAME }} exists!!!"
                      echo "#####################################################"  
                      $l = az storage blob exists --account-name ${{ parameters.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --container-name ${{ parameters.STORAGEACCOUNTCONTAINERNAME }} --name ${{ parameters.TFSTATEFILENAME }} --query "exists" --out tsv  
                        if ($l -eq "true") {
                          echo "#####################################################"
                          echo "Terraform State Blob ${{ parameters.TFSTATEFILENAME }} exists!!!"
                          echo "#####################################################"
                          az storage blob lease break --account-name ${{ parameters.STORAGEACCOUNTNAME }} --account-key $(az storage account keys list -g ${{ parameters.RGNAME }} -n ${{ parameters.STORAGEACCOUNTNAME }} --query [0].value -o tsv) --container-name ${{ parameters.STORAGEACCOUNTCONTAINERNAME }} --blob-name ${{ parameters.TFSTATEFILENAME }}                           
                          }
                        else {
                          echo "#####################################################"
                          echo "Terraform State Blob ${{ parameters.TFSTATEFILENAME }} DOES NOT EXISTS in ${{ parameters.STORAGEACCOUNTCONTAINERNAME }}"
                          echo "#####################################################"
                          exit 1
                          }
                    }
                    else {
                      echo "#####################################################"
                      echo "Storage Account Container ${{ parameters.STORAGEACCOUNTCONTAINERNAME }} DOES NOT EXISTS in STORAGE ACCOUNT ${{ parameters.STORAGEACCOUNTNAME }}!!!"
                      echo "#####################################################"
                      exit 1
                    }
                }
                else {
                  echo "#####################################################"
                  echo "Storage Account ${{ parameters.STORAGEACCOUNTNAME }} DOES NOT EXISTS!!!"
                  echo "#####################################################"
                  exit 1
                }              
            }
            else {
              echo "#####################################################"
              echo "Resource Group ${{ parameters.RGNAME }} DOES NOT EXISTS!!!"
              echo "#####################################################"
              exit 1
            }

```

| __OBJECTIVE OF TERRAFORM CODE SNIPPET:-__ |
| --------- |
| Create a Resource Group and User Assigned System Managed Identity. |
| The Purpose of the Terraform Code Snippet is to Reproduce the Issue, by Locking Terraform State File . |


| __TERRAFORM (main.tf):-__ |
| --------- |

```
terraform {
  required_version = ">= 1.2.3"

   backend "azurerm" {
    resource_group_name  = "tfpipeline-rg"
    storage_account_name = "tfpipelinesa"
    container_name       = "terraform"
    key                  = "TF-LEASE/BreakLease.tfstate"
  }
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.2"
    }   
  }
}
provider "azurerm" {
  features {}
  skip_provider_registration = true
}

```

| __TERRAFORM (usrmid.tf):-__ |
| --------- |

```
## Azure Resource Group:-
resource "azurerm_resource_group" "rg" {
 name     = var.rg-name
 location = var.rg-location
}

## Azure User Assigned Managed Identities:-
resource "azurerm_user_assigned_identity" "az-usr-mid" {
  
 name                = var.usr-mid-name
 resource_group_name = azurerm_resource_group.rg.name
 location            = azurerm_resource_group.rg.location
  
 depends_on          = [azurerm_resource_group.rg]
 }

```

| __TERRAFORM (variables.tf):-__ |
| --------- |

```
variable "rg-name" {
  type        = string
  description = "Name of the Resource Group"
}

variable "rg-location" {
  type        = string
  description = "Resource Group Location"
}

variable "usr-mid-name" {
  type        = string
  description = "Name of the User Assigned Managed Identity"
}

```

| __TERRAFORM (usrmid.tfvars):-__ |
| --------- |

```
rg-name         = "AMTest100"
rg-location     = "West Europe"
usr-mid-name    = "AMUSRMID100"

```

| __HOW TO LOCK TERRAFORM STATE FILE:-__ |
| --------- |

Run the Terraform Apply Command manually in your local System as mentioned below:-

```
terraform apply --var-file="usrmid.tfvars"

```

When Prompted for "Yes", Press __Control + C__ to terminate the Execution.

| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/41fqa716hcaxk1x3kif4.png) |
| --------- |
 
Next time, when you Re-Execute the Command, It will Inform the User that Terraform State File is in Locked State. 

| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mu6e9nxnalx79ay7c6xi.png) |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ghbcvd4lnz352xsumij7.JPG) | 


__NOW ITS TIME TO TEST !!!...__

| __TEST CASES:-__ |
| --------- |

| __TEST CASE #1: WRONG OR NO RESOURCE GROUP NAME PROVIDED:-__ |
| --------- |
| __DESIRED OUTPUT: PIPELINE WILL FAIL DISPLAYING CUSTOM MESSGAGE.__ |
| __PIPELINE RUN:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g3cjf17g7fcn3i6s0f8r.png) |
| __TEST CASE #2: WRONG OR NO STORAGE ACCOUNT NAME PROVIDED:-__ |
| __DESIRED OUTPUT: PIPELINE WILL FAIL DISPLAYING CUSTOM MESSGAGE.__ |
| __PIPELINE RUN:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rskfr9gk2s9cb3ekrg9r.png) |
| __TEST CASE #3: WRONG OR NO STORAGE ACCOUNT CONTAINER NAME PROVIDED:-__ |
| __DESIRED OUTPUT: PIPELINE WILL FAIL DISPLAYING CUSTOM MESSGAGE.__ |
| __PIPELINE RUN:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jiu7jbbr1fu4tmbbhhfd.png) |
| __TEST CASE #4: WRONG OR NO TERRAFORM STATE FILE NAME PROVIDED:-__ |
| __DESIRED OUTPUT: PIPELINE WILL FAIL DISPLAYING CUSTOM MESSGAGE.__ |
| __PIPELINE RUN:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bvpnzuxblfkxdb2zgqyr.png) |
| __TEST CASE #5: TERRAFORM STATE FILE FOUND == LEASED:-__ |
| __DESIRED OUTPUT: PIPELINE WILL RUN SUCCESSFULLY BREAKING THE TERRAFORM BLOB LEASED STATE.__ |
| __PIPELINE RUN:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/llj9qb2m07yfrc295sjh.JPG) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bpekztfh3hv6g8ny65ub.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/snr2j84o2dh2r85ve3mp.png) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ls2tasz7tgonthndy7p8.JPG) |
| __TEST CASE #6: TERRAFORM STATE FILE FOUND != LEASED:-__ |
| __DESIRED OUTPUT: PIPELINE STILL EXECUTES SUCCESSFULLY WITHOUT ALTERING THE PRESENT STATE.__ |
| __PIPELINE RUN:-__ |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5ygtnpanunmtil2r1ysk.JPG) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h7m4jxm9f561ncxqlwfm.JPG) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tn0n2cvp5d42r6c79uf9.JPG) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mkfo60add6klcwjcnc79.JPG) |


Hope You Enjoyed the Session!!!

Stay Safe | Keep Learning | Spread Knowledge
