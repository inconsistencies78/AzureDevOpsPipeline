# AzureDevOpsPipeline
how to use Azure DevOps Pipelines with Terraform and credentials in a keyvault

## to do
create a Azure Pipeline to deploy a simple terraform structure with:

- provider.tf
- main.tf
- variables.tf

credentials for provider.tf shall be in a Azure keyvault as secrets

# create service principal, keyvault, secrets and necessary ressources
## create service principal name (SPN) and client secret
login into your azure account
> az login

create SPN
> az ad sp create-for-rbac --name AzureDevOps --role="Contributor"

copy values for later

## create resource group
> az group create --name AzureDevOps --location "northeurope"

## create storage-account (to store tfstate-file)
> az storage account create --resource-group AzureDevOps --name sa01azuredevops --sku Standard_LRS --encryption-services blob

## get storage account key
> az storage account keys list --resource-group AzureDevOps --account-name sa01azuredevops --query [0].value -o tsv

copy value for later

## create storage container
> az storage container create --name container01-azuredevops --account-name sa01azuredevops --account-key "storage-account-access-key"

## create keyvault
> az keyvault create -n keyvault-devops01 -g AzureDevops -l "northeurope"

## add the storage account access key to keyvault
> az keyvault secret set --vault-name "keyvault-devops01" --name "sa01-azdo-accesskey" --value "storage-account-access-key"

## add the service principal password to keyvault
> az keyvault secret set --vault-name "keyvault-devops01" --name "spn-azuredevops-password" --value "service-principal-password"

## allow the service principal name access to keyvault with permissions "get" and "list"
> az keyvault set-policy --name "keyvault-devops01" --spn "0db8ed92-a623-4c37-a03c-42d8398f7c3" --secret-permissions get list

# create and configure Azure DevOps
go-to dev.azure.com and create (if necessary) an organization and a project
## create repository
Click your new Team Project and select Repos. Click Initialize to create a blank repository

## create variable group
create variable group to store values and make available accross multiple pipelines

> az pipelines variable-group create --name "Terraform-BuildVariables" --authorize true --organization https://dev.azure.com/your-organization/ --project "your-project-name" --variables foo=bar

The foo=bar variable isn’t used, but a single variable is required to first create the variable group.
for further inforamtions about this check 
https://adamtheautomator.com/azure-devops-pipeline-infrastructure/#yaml-pipeline-review

## connect the variable group to keyvault
pipelines -> library -> toggle “Link secrets from an Azure key vault as variables.”
select subscription
authorize subscription
authorize keyvault

add variables

choose secrets

# adding the terraform code to Azure DevOps
use variables in provider.tf
  subscription_id = var.subscription_id
  client_id       = var.client_id
  client_secret   = var.client_secret
  tenant_id       = var.tenant_id

## install Terraform Azure DevOps extension
https://marketplace.visualstudio.com/items?itemName=ms-devlabs.custom-terraform-tasks

# create Azure DevOps pipeline
go to dev.azure.com
-> create pipeline

## choose repo
configure pipeline
starter pipeline
customize pipeline

## YAML Configuration File
- Trigger: To make the Pipeline automatically run, we need to define a trigger. The trigger below causes the DevOps Pipeline to run when a commit is done on the Master Branch of our repo.
- Paths: ‘By default, if you don’t specifically include or exclude files or directories in a CI build, the pipeline will run when a commit is done on any file.’2 Since my project contains other files not relating to this Terraform project, I only want it to run when changes are done in my two terraform files ( variables.tf and main.tf)
- Pool: The VM that is going to run my code and deploy my infrastructure
- Group: The Variable Group containing my secrets and their corresponding values. These will automatically be imported in
- subscription_id: The subscription ID I am deploying my resources to
- application_id: The application ID of my service principal name
- tenant_id: The ID of my Azure tenant
- storage_accounts: The Storage Account that houses my Storage Container that contains my state file
- blob_storage: The Storage Container that will house my state file
- state_file: The name of my Terraform state file
- sa-resource_group: The Resource Group that my Storage Account is in

### configuration yaml file
trigger:
  branches:
    include:
      - master
  paths:
    include:
      - /Azure_DevOps/Terraform/variables.tf
      - /Azure_DevOps/Terraform/main.tf

pool:
  vmImage: "ubuntu-latest"

variables:
  - group: Terraform-BuildVariables
  - name: subscription_id
    value: "5bc98db6-521c-1ed5-a023-8fe8r41b0460"
  - name: application_id
    value: "034j3d92-a6123-4c56-a03c-4348566f167c3"
  - name: tenant_id
    value: "63546b2c9-54e9-4f5e-9851-f00c4j323dc1f"
  - name: storage_accounts
    value: "sa01azuredevops"
  - name: blob_storage
    value: container01-azuredevops
  - name: state_file
    value: tf-statefile.state
  - name: sa-resource_group
    value: AzureDevOps

steps:
  - task: ms-devlabs.custom-terraform-tasks.custom-terraform-installer-task.TerraformInstaller@0
    displayName: 'Install Terraform'
  - script: terraform version
    displayName: Terraform Version
  - script:  az login --service-principal -u $(application_id) -p $(spn-azuredevops-password) --tenant $(tenant_id)
    displayName: 'Log Into Azure'
  - script: terraform init -backend-config=resource_group_name=$(sa-resource_group) -backend-config="storage_account_name=$(storage_accounts)" -backend-config="container_name=$(blob_storage)" -backend-config="access_key=$(sa01-azdo-accesskey)" -backend-config="key=$(state_file)"
    displayName: "Terraform Init"
    workingDirectory: $(System.DefaultWorkingDirectory)/Azure_DevOps/Terraform
  - script: terraform plan -var="client_id=$(application_id)" -var="client_secret=$(spn-azuredevops-password)" -var="tenant_id=$(tenant_id)" -var="subscription_id=$(subscription_id)" -var="admin_password=$(vm-admin-password)" -out="out.plan"
    displayName: Terraform Plan
    workingDirectory: $(System.DefaultWorkingDirectory)/Azure_DevOps/Terraform
  - script: terraform apply out.plan
    displayName: 'Terraform Apply'
    workingDirectory: $(System.DefaultWorkingDirectory)/Azure_DevOps/Terraform


-> save and run
-> view -> permit

:-)

# references
https://www.thelazyadministrator.com/2020/04/28/deploy-and-manage-azure-infrastructure-using-terraform-remote-state-and-azure-devops-pipelines-yaml/

https://adamtheautomator.com/azure-devops/#yaml-pipeline-review

