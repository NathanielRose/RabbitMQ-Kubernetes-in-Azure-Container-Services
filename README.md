# Deploying a RabbitMQ Application into an Kubernetes orchestrated Azure Container Service Cluster

In this example we will be deploying a RabbitMQ application into a Kubernetes orcherstrated cluster on Azure Container Service.

> Azure Container Service makes it simpler for you to create, configure, and manage a cluster of virtual machines that are preconfigured to run containerized applications. It uses an optimized configuration of popular open-source scheduling and orchestration tools. This enables you to use your existing skills, or draw upon a large and growing body of community expertise, to deploy and manage container-based applications on Microsoft Azure.

<img src="https://rtwrt.blob.core.windows.net/post3-rabbitmq/Rabbitmq.jpg" width="400">



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

On Windows run [Bash](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide) and follow this [guide](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/) to create a new SSH key and add it to the ssh-agent

``` Powershell

ssh-keygen -t rsa -b 4096 -C "youremail@address.com"

Enter passphrase (empty for no passphrase):

Enter same passphrase again:

cat ~/.ssh/id_rsa.pub

```

Run the following command, using `--parameters` to set the path to the `azuredeploy.parameters.json` file. This command deploys the cluster in a resource group you create called `myResourceGroup` in the West US region.

```PowerShell

az login

az account set --subscription "mySubscriptionID"

az group create --name "myResourceGroup" --location "westus" 

az group deployment create -g "myResourceGroup" --template-uri "https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/101-acs-kubernetes/azuredeploy.json" --parameters @azuredeploy.parameters.json

```

Inside your Azure Resource group you will see all the resources that get provisioned within your subscription after deploying that ARM template.

<img src="https://rtwrt.blob.core.windows.net/post3-rabbitmq/img1.PNG" width="900">


## Using Helm to deploy a RabbitMQ Container

In this part we will connect to our ACS Kubernetes cluster and deploy a Rabbit MQ Helm Chart container.

Lets first connect to our newly deployed Azure Container Service pool. Start by installing kubectl. 

  ``` CommandPrompt
  C:\Users\naros\AppData\Roaming\Python\Python35\Scripts>az acs kubernetes install-cli

  C:\Users\naros\AppData\Roaming\Python\Python35\Scripts>kubectl get context
  the server doesn't have a resource type "context"
  ```
Lets set our kube config file to the appropriate one we just deployed

``` Powershell
az acs kubernetes get-credentials --resource-group=RabbitMQ-ACS --name=containerservice-RabbitMQ-ACS
```

If you run into an Authentication failed issue, open `bash` and navigate to the root file of your master in the pool. Grab the config file and copy it to your local device.

```Bash

nmrose@MININT-IQPVRH0:~/.ssh$ ssh -i id_rsa azureuser@rabbitmqacsmgmt.westus.cloudapp.azure.com
Enter passphrase for key 'id_rsa':

azureuser@k8s-master-CAAF9CF0-0:~$ cd .kube
azureuser@k8s-master-CAAF9CF0-0:~/.kube$ ls

scp azureuser@rabbitmqacsmgmt.westus.cloudapp.azure.com:/home/azureuser/.kube/config ~/.kube/config
```

Once completed run `kubectl cluster-info' and you will see details of your kubernetes cluster.

```
Kubernetes master is running at https://rabbitmqacsmgmt.westus.cloudapp.azure.com
Heapster is running at https://rabbitmqacsmgmt.westus.cloudapp.azure.com/api/v1/proxy/namespaces/kube-system/services/heapster
KubeDNS is running at https://rabbitmqacsmgmt.westus.cloudapp.azure.com/api/v1/proxy/namespaces/kube-system/services/kube-dns
kubernetes-dashboard is running at https://rabbitmqacsmgmt.westus.cloudapp.azure.com/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
```

Lets first deploy a simg Nginx server to the pool.

```Bash
kubectl get nodes
kubectl.exe run nginx --image nginx --port=80
```

It will take a few moments to have the nginx pod ready. Once ready use `kubectl get pods` to get the status.

Now lets use helm to deploy a RabbitMQ container. Helm is a tool for managing Kubernetes charts. Charts are packages of pre-configured Kubernetes resources.

First we need to get the Helm libraries. Then lets use the default rabbitMQ chart for this example.

```Bash
helm init

helm installstable/rabbitmq

```