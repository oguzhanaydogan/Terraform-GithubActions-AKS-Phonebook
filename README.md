# GitHub Actions Terraform AKS Deployment and Deploying an App to AKS
This project sets up a CI/CD pipeline using GitHub Actions to deploy infrastructure using Terraform and deploy Phonebook Application to that infrastructure. The pipeline will be triggered on every push to the main branch and every pull request to the main branch.

## Setting Credentials
Terraform needs Azure Credentials to create the infrastructure. We need to provide these values in environment for Terraform to look up.
- ARM_SUBSCRIPTION_ID
- ARM_TENANT_ID
- ARM_CLIENT_ID
- ARM_CLIENT_SECRET
- AZURE_SERVICE_PRINCIPAL 

To get these credentials we use this command;
```
az ad sp create-for-rbac --sdk-auth --role="Contributor" --scopes="/subscriptions/<subscription_id>"
```

Terraform also needs GitHub Token to create the Variables in GitHub repository. We provide the token securely by defining it in the GitHub Actions Secrets as `GH_TOKEN`. We assign this value in the pipeline environment section to `GITHUB_TOKEN` with:
```
GITHUB_TOKEN: ${{ secrets.GH_TOKEN }}
```
## Terraform Configuration
The Terraform configuration file deploys the following resources:
- azurerm_resource_group: Creates an Azure resource group named ${var.name} in the location ${var.location}.
- azurerm_kubernetes_cluster: Creates an Azure Kubernetes Service cluster named ${var.name} in the location ${var.location} with a default node pool containing one node.
- data.azurerm_lb: Retrieves the Azure Load Balancer used by the AKS cluster.
- data.azurerm_lb_backend_address_pool: Retrieves the backend address pool used by the Load Balancer.
- azurerm_lb_probe: Creates two load balancer probes on port 30001 and 30002.
- azurerm_lb_rule: Creates two load balancer rules for ports 30001 and 30002.

Since GitHub Actions Pipeline uses an ephemeral agent we need to define a backend to keep our `terraform.tfstate`. We configure our backend to Azure Blob Container with following code 
```
  backend "azurerm" {
    resource_group_name  = "sshkey"
    storage_account_name = "ccseyhan"
    container_name       = "test"
    key                  = "terraform.tfstate"
  }
```

To use later in the pipeline we define multiple `github_actions_environment_variable`
- `NODERG` = azurerm_kubernetes_cluster.aks.node_resource_group
- `AKSRG_NAME`
- `AKS_NAME`

## GitHub Actions Configuration
The pipeline is configured using GitHub Actions. The configuration file main.yml is located in the .github/workflows directory. The pipeline is triggered by pushes to the main branch and pull requests against the main branch.

Since we have our Terraform configuration files in a dedicated folder, we need to define this path in the job environment for the steps which need to access to this folder to run 
```
env:
    working-directory: Terraform_infra/
``` 

In a similar fashion we need to define `k8s` path to apply our Kubernetes Manifest Files
```
env: 
    working-directory: k8s/
``` 

Our application needs the ports `30001-30002` open to be accessed. Since the NSG name of the AKS assigned randomly by Azure we need to get its name and add a rule to it with the following step
```
- name: 'Create Nsg Rule'
  working-directory: ${{ env.working-directory }}
  run: |
    nsg_name=$(az network nsg list --resource-group ${{ vars.NODERG }} --query "[?contains(name, 'aks')].[name]" --output tsv)
    az network nsg rule create --nsg-name $nsg_name --resource-group ${{ vars.NODERG }} \
     --name open30001 --access Allow --priority 100 --destination-port-ranges 30001-30002
```