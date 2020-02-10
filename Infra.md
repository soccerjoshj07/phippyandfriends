# Infra Setup

## Environment Variables

Set up the environment variables the scripts below will use

```bash
LOCATION=EastUS2
RG=rg-phippyaks-demo2
ACR=aciphippyaksdemo
NAME=aks-phippyaksdemo
```

## Container registry

We will use Azure Container Registry (ACR) to store both your Docker images and your Helm charts. For that you can run the commands below from Azure Cloud Shell:

```bash
# Create a resource group $RG on a specific location $LOCATION (for example eastus) which will contain the Azure services we need 
$ az group create -l $LOCATION -n $RG
# Create an ACR registry $ACR
$ az acr create -n $ACR -g $RG -l $LOCATION --sku Basic
```

## Kubernetes cluster

We will leverage Azure Kubernetes Service (AKS). For that, you can run the commands below from Azure Cloud Shell:

```bash
# Setup of the AKS cluster
### latest version = preview, so not passing that in.
#$ latestK8sVersion=$(az aks get-versions -l $LOCATION --query 'orchestrators[-1].orchestratorVersion' -o tsv)
#$ az aks create -l $LOCATION -n $NAME -g $RG --generate-ssh-keys -k $latestK8sVersion -s Standard_B2s
###
$ az aks create -l $LOCATION -n $NAME -g $RG --generate-ssh-keys -s Standard_B2s -c 2
# Once created (the creation could take ~10 min), get the credentials to interact with your AKS cluster
$ az aks get-credentials -n $NAME -g $RG
# Setup the phippyandfriends namespace, you will deploy later some apps into it
$ kubectl create namespace phippyandfriends
$ kubectl create clusterrolebinding default-view --clusterrole=view --serviceaccount=phippyandfriends:default
```

## Azure Roles and Permissions

We need to assign 3 specific service principals with specific Azure Roles that need to interact with our ACR and our AKS

``` bash
# 1. Assign acrpull role on our ACR to the AKS-generated service principal, the AKS cluster will then be able to pull images from our ACR
$ ACR_ID=$(az acr show -n $ACR -g $RG --query id -o tsv)
$ az aks update -g $RG -n $aks --attach-acr $ACR_ID
# 2. Create a specific Service Principal for our Azure DevOps pipelines to be able to push and pull images and charts of our ACR
$ registryPassword=$(az ad sp create-for-rbac -n $ACR-push --scopes $ACR_ID --role acrpush --query password -o tsv)
# Important note: you will need this registryPassword value later in this blog article in the Create a Build pipeline and Create a Release pipeline sections
$ echo $registryPassword
# 3. Create a specific Service Principal for our Azure DevOps pipelines to be able to deploy our application in our AKS
$ AKS_ID=$(az aks show -n $aks -g $RG --query id -o tsv)
$ aksSpPassword=$(az ad sp create-for-rbac -n $aks-deploy --scopes $AKS_ID --role "Azure Kubernetes Service Cluster User Role" --query password -o tsv)
# Important note: you will need this aksSpPassword value later in this blog article in the Create a Release pipeline section
$ echo $aksSpPassword
```
helm upgrade --namespace phippyandfriends --install --force --set ingress.enabled=false,image.repository=aciphippyaksdemo.azurecr.io/soccerjoshjphippyandfriends,image.tag=20200203.19 --wait phippyandfriends-website ~/Repos/phippyandfriends/parrot/charts/parrot


helm template /home/vsts/work/1/charts --namespace phippyandfriends --set ingress.enabled=false= --set image.repository=aciphippyaksdemo.azurecr.io/soccerjoshjphippyandfriends= --set image.tag=20200203.24