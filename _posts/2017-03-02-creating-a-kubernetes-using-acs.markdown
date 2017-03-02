---
layout: post
title: "Creating a Kubernetes cluster using Azure Container Service"
date: 2017-03-02 12:00:00 +0100
categories: docker azure acs kubernetes
---
In my [previous post](2016-10-01-creating-a-docker-swarm-in-azure.markdown) Docker Machine was used to create a Docker swarm in Azure.

In this post we will provision a Kubernetes cluster in Azure using [Azure Container Service](https://azure.microsoft.com/en-gb/services/container-service/) and the Azure CLI 2.0.

# Requirements
The following requirements are needed to for creating the swarm:

- [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- An Azure account

Ensure the Azure CLI 2.0 is installed and that you are logged in and you have selected the subscription you want to use.  You can use the CLI to view the account/subscription that is logged in and selected:
```bash
az account show
```
If you are not logged in follow the instructions [here](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli) to login. If you have multiple subscriptions follow the instructions [here](https://docs.microsoft.com/en-us/cli/azure/manage-azure-subscriptions-azure-cli) to set the default subscription to use.

# Cluster Creation
Creating a cluster is made up of a number of steps:

* [Planning](#planning)
* [Cluster creation](#cluster-creation)
* [Validate the cluster](#validate-the-cluster)

## Planning
Like with the Docker Machine approach to creating a Docker Swam the first step is deciding how many *master* and *worker* nodes there will be.  Specific advice in relation to Kubernetes is hard to find in the official documentation but as a rough guide:
**Type**|**Advice**
  -------------|-------------
  Worker|For production environments more than 1 node is recommended.
  Master|For production environments more than 1 node is recommended.

##Cluster creation
1) Create a resource group for your swarm:
```bash
az group create -n dockercoins -l "westeurope"
```

2) Create the cluster in the resource group using the following command:
```bash
az acs create -n dockercoins -g dockercoins -d rmc-dockercoins --orchestrator-type kubernetes
```
This will create a cluster with:

* The default number of master and worker nodes. If you wanted to use a specific number you could use the *--agent-count* and *--master-count* command line options.
* With a name of *dockercoins*
* In the *dockercoins* resource group
* With a DNS prefix of *rmc-dockercoins*
* Using the SSH keys that already exist in the c:\users\\<user>\\\.ssh folder. If there are no SSH keys already then you can use the *--generate-ssh-keys* command line option.

A full list of the command line options can be found [here](https://docs.microsoft.com/en-us/cli/azure/acs#create). 

NOTE: The main feature of Azure Container Service is that it can provision container clusters using a number of orchestration engines. In this post we are using Kubernetes but we could also use Docker Swarm or DC/OS. *Docker Swarm Mode* isn't currently supported.

3) Once the command has completed you will see a number of Azure artefacts provisioned within the resource group:
![resource group](/images/swarmacs/Resourcegroup.PNG)

4) The next step is to install the *kubectl* command line utility that is the main method for interacting with the Kubernetes cluster. This can be done using the following command line:

```bash
 az acs kubernetes install-cli --install-location C:\Dev\Tools
```

This will download *kubectl* to the location specified in the *--install-location* command line argument. If you don't specify this it will try to save it to c:\program files and you will need to run the command as administrator.

5) Download the Kubernetes configuration file for the cluster for use with *kubectl* using the following:

```bash
az acs kubernetes get-credentials --dns-prefix=rmc-dockercoins --location=westeurope --user azureuser
```

This will create the required configuration files in the C:\Users\\<user>\\.kube folder.

##Validate the cluster
1) To test communication with the cluster run the following command to get a list of the nodes in the cluster :
```bash
kubectl get nodes
```
The output will show showing similar to the following:
![fqdn](/images/swarmacs/getnodes.png)

NOTE: The *master* node has a status of **SchedulingDisabled** which means that no pods will be scheduled on the node. This is recommended for production systems as the master nodes should only be used to administer and schedule work on the worker/agent nodes.

2) Run the following command to start the local proxy for connecting to the Kuberenetes dashboard:

```bash
kubectl proxy
```

 Then open a browser and navigate [here](http://127.0.0.1:8001/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#/workload?namespace=default).

# Next Steps

Now the Kubernetes cluster is running in Azure we can deploy applications to it. It an following post we will:

* Deploy a sample application to the cluster
* Expose a Web UI running in the cluster publicly



