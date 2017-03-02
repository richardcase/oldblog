---
layout: post
title:  "Creating a Docker swarm in Azure"
date:   2016-10-01 08:12:45 +0100
categories: docker azure
---
The release of Docker Engine v1.12 introduced swarm mode for creating and managing a cluster of Docker Engines (which is referred to as a swarm). This replaces Docker Swarm which was previously separate to the Docker Engine.

A Swarm is a collection (cluster) of nodes running Docker Engine following a decentralized design where services can be deployed. Swarm mode includes scaling, service discovery, desired state and many other features. For an overview of Swarm mode see the docs [here](https://docs.docker.com/engine/swarm/).

By using [Docker Machine](https://docs.docker.com/machine/) and the [Azure driver](https://docs.docker.com/machine/drivers/azure/), a Docker swarm can be easily created in Azure. This involves running a small number of Docker Machine commands to create the individual nodes of the Swarm. We can then run Docker CLI commands to create the swarm.

# Requirements

The following requirements are needed to create a Docker swarm in Azure.

- [Docker v1.12 or later](https://www.docker.com/products/overview)
- [Docker Machine](https://docs.docker.com/machine/install-machine/) 
- An Azure account

If you don’t have a Azure account yet, no problem. Head over to their website and [sign up for a free trial](https://azure.microsoft.com).


# Swarm Creation

Creating a Docker swarm is made of a number of steps:

* [Planning](#environment-setup)
* [Node Creation](#node-creation)
* [Create the swarm](#create-the-swarm)
* [Add Manager Nodes](#add-manager-nodes)
* [Add Worker Nodes](#add-worker-nodes)
* [Validate the Cluster](#validate-the-cluster)


## Planning

1) Decide how many *manager* and *worker* nodes you want in your swarm. The following table contains some advice:

 **Type**|**Advice**
  -------------|-------------
  Worker|For production environments more than 1 node is recommended.
  Manager|For production environments more than 1 node is recommended. Additionally, due to the use of the Raft consensus algorithm in the manager nodes its recommended that there are a odd number of manager nodes. 

NOTE: By default the manager nodes will be used to run tasks in addition to the worker nodes. In a production environment you would only want the workers to run tasks and so you'd need to *drain* the manager nodes.

NOTE: More Information on Raft can be found [here](http://thesecretlivesofdata.com/raft/).

## Node creation

1) Create the manager nodes by running the following command once for each manager to create:

{% highlight shell %}
docker-machine create -d azure --azure-subscription-id xxxx --azure-image "canonical:UbuntuServer:16.04.0-LTS:latest"  manager1
{% endhighlight %}

NOTE: Docker machine will use *manager1* as the identifier for this manager node. Therefore, when creating multiple manager nodes make sure this name is unique.

NOTE: The *azure-subscription-id* command line option is required and you must replace xxxx with your subscription id. For a full list of the supported command line options see the [docs](https://docs.docker.com/machine/drivers/azure/).

When you run the a Docker Machine command for the first time using the Azure driver it will ask you to authenticate:
![docker-machine azure auth](/images/swarminazure/azure_driver_auth.png)
Open a browser and navigate to the link and enter the unique authentication code provided. You will not be asked to authenticate again for 2 weeks.

2) Create the worker nodes by running the following command once for each worker to create:
{% highlight shell %}
docker-machine create -d azure --azure-subscription-id xxxx --azure-image "canonical:UbuntuServer:16.04.0-LTS:latest"  worker1
{% endhighlight %}

The notes from step 1) apply to the worker nodes as well.

3) Once the nodes have been created run the following command to see a list of all the Docker hosts that have been created:
{% highlight shell %}
docker-machine ls
{% endhighlight %}

The output will be similar to the following:
![docker-machine ls](/images/swarminazure/machine_ls.png)

## Create the swarm

1) Set the Docker environment variables for the first of the manager nodes by first running:
{% highlight shell %}
docker-machine env manager1
{% endhighlight %}

Depending on your OS the command will output a command that can be used to set the required environment variables. For example:
{% highlight shell %}
eval $(docker-machine env manager1)
{% endhighlight %}

2) Create the swarm by running the following command:
{% highlight shell %}
docker swarm init
{% endhighlight %}

The output of the command will contain the command that will be used later to join worker nodes to the swarm. Save the command for later use:

![swarm init](/images/swarminazure/swarm_init.png)

NOTE: Depending on how many IP addresses the node has you may need to specify which address to use by using the *--advertise-addr* command line option. For a full list of the options see the [docs](https://docs.docker.com/engine/reference/commandline/swarm_init/).

## Add manager nodes

1) Whilst the Docker environment variables are still set to the first manager node run the following command:
{% highlight shell %}
docker swarm join-token manager
{% endhighlight %}

The output will contain the command that will be required to join additional manager nodes to the swarm.

2) Set the Docker environment variables for the manager node to join to the existing swarm by first running:
{% highlight shell %}
docker-machine env manager2
{% endhighlight %}

Depending on your OS this will output a command that can be used to set the required environment variables. For example:
{% highlight shell %}
eval $(docker-machine env manager2)
{% endhighlight %}

3) Run the command that was output in step 1). For example:
{% highlight shell %}
docker swarm join \
    --token SWMTKN-1-2lptlwq6p1ddt0t6qpil2p904qlmrwnrik7f5a2c0gdvvs7hn4-cuxeu68q9ec159axrciz6wbqf \
    192.168.0.4:2377
{% endhighlight %}

The output of the command should indicate that the node joined the swarm as a manager.

4) Repeat steps 2) & 3) for each of the remaining manager nodes that need to join the swarm.

## Add worker nodes

1) Set the Docker environment variables for the worker node to join to the existing swarm by first running:
{% highlight shell %}
docker-machine env worker1
{% endhighlight %}

Depending on your OS this will output a command that can be used to set the required environment variables. For example:
{% highlight shell %}
eval $(docker-machine env worker1)
{% endhighlight %}

2) Run the command that was output in step 2) of the [Create the swarm](#create-the-swarm) section. For example:
{% highlight shell %}
docker swarm join \
    --token SWMTKN-1-2lptlwq6p1ddt0t6qpil2p904qlmrwnrik7f5a2c0gdvvs7hn4-138uv6z599o1q9gki9wzwumn7 \
    192.168.0.4:2377
{% endhighlight %}

The output of the command should indicate that the node joined the swarm as a worker.

3) Repeat steps 1) & 2) for each of the remaining worker nodes that need to join the swarm.

## Validate the cluster

1) Run the following command to list the nodes of the swarm along with their status:
{% highlight shell %}
docker node ls
{% endhighlight %}

You should see output similar to the following:
![node ls](/images/swarminazure/node_ls.png)
This shows various information including the status of each node and which manager has been elected the leader.

2) Logon to the [Azure portal](https://portal.azure.com/) and navigate to the resource groups. If you used the *--azure-resource-group* command line option then you will see a new resource group with the name you supplied otherwise it will be called *docker-machine*.
![azure resource group list](/images/swarminazure/resource_groups.png)

If you click on the resource group you will see that a number of resources have been created in addition to the actual virtual servers:
![azure resource group overview](/images/swarminazure/resource_group_overview.png)

# Next Steps 
Now you have a working Docker swarm running in Azure you are ready to start managing and deploying services to the swarm. Its recommended that you follow the swarm mode tutorial from this [starting point](https://docs.docker.com/engine/swarm/manage-nodes/).

In subsequent posts we will be covering:

* Deploying a solution to Docker swarm.
* Creating a stretch Docker swarm.
* Productionalizing your Docker swarm. 