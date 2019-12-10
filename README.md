# Deploy basic AKS Cluster with Azure CLI

This folder contains the assets needed to deploy a basic AKS Cluster in a Microsoft Azure subscription.

The assets are designed to support the module <em>Deploying Azure Kubernetes Service</em> <strong>Manage Kubernetes Infrastructure</strong>.

The deployment steps are documented in the Bash script [deployAks.sh](scripts/deployAks.sh). This script is not designed to be a standalone asset, but rather a step-by-step process to show some of the various options for deploying a simple AKS cluster.

This demo also uses an ARM template to deploy a <strong>Log Analytics workspace</strong> in the solution Resource Group for AKS cluster monitoring. Azure CLI can do this automatically for you, but in this case you relinquish control over configuration options such as the workspace name. The ARM template is located in the [arm/logAnalytics](arm/logAnalytics) folder.

## Requirements

The script assumes that you are running a system with [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed. You should also have authenticated against an Azure environment using `az login` with credentials which have sufficient permissions in the Azure AD tenant to create new service principals and Owner permissions at the subscription scope.

The steps in the deployment script can be run on any system which support running Bash scripts: [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10) on Windows 10, Linux or MacOS. You can also execute all the steps using an [Azure Cloud Shell](https://azure.microsoft.com/en-au/features/cloud-shell/) session, either directly via the Azure Portal, using the [Azure Account](https://marketplace.visualstudio.com/items?itemName=ms-vscode.azure-account) extension in VSCode or using the new [Windows Terminal](https://github.com/microsoft/terminal).



# Steps Involved to Provison Basic AKS Cluster.

## Step1: Open Azure Cloud Shell and git clone this Repository.
## Step2: Check Azure Version: az --version
## Step3: ls -la
## Step4: cd clouddrive # this folder enables persistent cloud storage in Azure(more than 20 minutes)
## Step5: mkdir git > cd git >mkdir demo> cd demo and git clone https://github.com/SauravSrivastav/DeployBasicAKSCluster.git
## Step6: cd DeployBasicAKSCluster
## Step7: Check Resource Provider registration:

# namespace='Microsoft.ContainerService'#
if [ "$(az provider show --namespace ${namespace} | jq -r .registrationState)" != 'Registered' ]
then
      az provider register --namespace ${namespace} --verbose
else
      echo "Namespace \"${namespace}\" is already registered."
fi

## Step8: Create Resource Group

groupName='demo-aks'
groupLocation='East US'
group=$(az group create --name ${groupName} --location "${groupLocation}" --verbose)

## Step9: Deploy Log Analytics Workspace

solution='logAnalytics'
templatePath='../arm'
templateFile="${templatePath}/${solution}/azureDeploy.json"

timestamp=$(date -u +%FT%TZ | tr -dc '[:alnum:]\n\r')
name="$(echo $group | jq .name -r)-${timestamp}"
deployment=$(az group deployment create --resource-group $(echo $group | jq .name -r) --name ${name} --template-file ${templateFile} --verbose)

## Step10: Deploy AKS environment
clusterName='demoCluster'

## Step11: Create Service Principal
spName=sp-aks-${clusterName}
sp=$(az ad sp create-for-rbac --name ${spName})

## Step12: Deploy AKS Cluster

logAnalyticsId=$(echo $deployment | jq .properties.outputs.workspaceResourceId.value -r)

az aks create \
    --resource-group $(echo $group | jq .name -r) \
    --location $(echo $group | jq .location -r) \
    --name ${clusterName} \
    --service-principal $(echo $sp | jq .appId -r) \
    --client-secret $(echo $sp | jq .password -r) \
    --node-count 1 \
    --node-vm-size Standard_DS1_v2 \
    --enable-addons monitoring \
    --generate-ssh-keys \
    --workspace-resource-id ${logAnalyticsId} \
    --disable-rbac \
    --verbose


## Step13: Get AKS Credentials
az aks get-credentials --resource-group $(echo $group | jq .name -r) --name ${clusterName}



