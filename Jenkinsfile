pipeline {
    agent any
    tools {
        terraform 'terraform'
    }

    stages {
        stage('Create Infrastructure for the App') {
            steps {
                sh 'az login --identity'
                dir('/var/lib/jenkins/workspace/Kubernetes-Phonebook-Azure/Terraform_infra'){
                    echo 'Creating Infrastructure for the App on AZURE Cloud'
                    sh 'terraform init'
                    sh 'terraform apply --auto-approve'
                }
            }
        }

        stage('Connect to AKS and set NSG permissions') {
            steps {
                dir('/var/lib/jenkins/workspace/Kubernetes-Phonebook-Azure/Terraform_infra'){
                    echo 'Injecting Terraform Output into connection command'
                    script {
                        env.AKS_NAME = sh(script: 'terraform output -raw aks_name', returnStdout:true).trim()
                        env.RG_NAME = sh(script: 'terraform output -raw rg_name', returnStdout:true).trim()
                        env.NODERG = sh(script: 'terraform output -raw noderg', returnStdout:true).trim()
                        env.NSG_NAME = sh(script: "az network nsg list --resource-group ${NODERG} --query \"[?contains(name, 'aks')].[name]\" --output tsv", returnStdout:true).trim()
                    }
                    sh 'az aks get-credentials --resource-group ${RG_NAME} --name ${AKS_NAME}'
                    sh 'az network nsg rule create --nsg-name ${NSG_NAME} --resource-group ${NODERG} --name open30001 --access Allow --priority 100 --destination-port-ranges 30001-30002'
                        



                }
            }
        }
        stage('Deploy K8s files') {
            steps {
                dir('/var/lib/jenkins/workspace/Kubernetes-Phonebook-Azure/k8s') {
                    sh 'kubectl apply -f .'
                }
            }
        }
        stage('Destroy the Infrastructure') {
            steps{
                timeout(time:5, unit:'DAYS'){
                    input message:'Do you want to terminate?'
                }
                dir('/var/lib/jenkins/workspace/Kubernetes-Phonebook-Azure/Terraform_infra'){
                    sh """
                    terraform destroy --auto-approve
                    """
                }
            }
        }
    }
}
