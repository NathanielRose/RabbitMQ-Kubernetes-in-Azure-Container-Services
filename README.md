# Deploying a RabbitMQ Application into an Kubernetes orchestrated Azure Container Service Cluster

In this example we will be deploying a RabbitMQ application into a Kubernetes orcherstrated cluster on Azure Container Service.

> Azure Container Service makes it simpler for you to create, configure, and manage a cluster of virtual machines that are preconfigured to run containerized applications. It uses an optimized configuration of popular open-source scheduling and orchestration tools. This enables you to use your existing skills, or draw upon a large and growing body of community expertise, to deploy and manage container-based applications on Microsoft Azure.

<li><a href="##Prerequisites">Prerequisites</a></li>
<li><a href="##Convert to Docker Image">Convert ASP.Net App to a Docker Image</a></li>
<li><a href="##Publish to Docker Hub">Publish Docker Image to a Container Registry</a></li>
<li><a href="##Deploy Service Fabric to Azure">Deploy a Service Fabric Cluster on Azure</a></li>
<li><a href="#install">Deploy Docker Image as a Stateless Service on Service Fabric</a></li>

## Prerequisites
  - Windows 10 - *With the Windows 10 anniversay edition you may need to turn off Secure Boot in the BIOS of your machine in order for the SF cluster to run locally*
  - [Visual Studio 2015 with Update 2 or Later](http://www.microsoft.com/web/handlers/webpi.ashx?command=getinstallerredirect&appid=MicrosoftAzure-ServiceFabric-VS2015)
  - [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
  - [Python 3.6](https://www.python.org/downloads/release/python-361/)
  - [Pika-Python](https://pypi.python.org/pypi/pika)
  - [Erlang](https://www.erlang.org/downloads)
  - [Kubectl](https://kubernetes.io/docs/tasks/kubectl/install/)
  - Windows Powershell
  - Visual Studio Team Services Account
  - Microsoft Azure Subscription
  - [Docker for windows](https://store.docker.com/editions/community/docker-ce-desktop-windows?tab=description)


## Create an ACS Cluster with Kubernetes

### Create a Service Principal

Create a service principal in Azure Active Directory for use in your Kubernetes cluster, Azure provides several methods.

The following example commands show you how to do this with the Azure CLI 2.0. You can alternatively create a service principal using Azure PowerShell, the classic portal, or other methods.

```Powershell
az login

az account set --subscription "mySubscriptionID"

az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/mySubscriptionID"
```

Confirm your service principal by opening a new shell and run the following commands, substituting in appId, password, and tenant:

```Powershell
az login --service-principal -u yourClientID -p yourClientSecret --tenant yourTenant

az vm list-sizes --location westus
```
> For more info on how to create a service principal check out this [document](https://github.com/Microsoft/azure-docs/blob/master/articles/container-service/container-service-kubernetes-service-principal.md). 

### Pass the service Principal client ID and client secret and deploy

Download the template parameters file `azuredeploy.parameters.json` included in this directory.

To specify the service principal, enter values for `servicePrincipalClientId` and `servicePrincipalClientSecret` in the file. (You also need to provide your own values for `dnsNamePrefix` and `sshRSAPublicKey`. The latter is the SSH public key to access the cluster.) Save the file.

