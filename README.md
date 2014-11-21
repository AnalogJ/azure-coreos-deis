#How to setup a Deis (Heroku-like PAAS) on Microsoft Azure using CoreOS

##Prerequsites

- Install and Configure the Azure CLI

    # If Node.js is installed on your system, use the following command to install the xplat-cli:
    sudo npm install azure-cli -g

    #To download the publish settings for your account, use the following command:
    azure account download

    #This will open your default browser and prompt you to sign in to the Azure Management Portal. After signing in, a .publishsettings file will be downloaded. Make note of where this file is saved.

    #Next, import the .publishsettings file by running the following command, replacing [path to .publishsettings file] with the path to your .publishsettings file:
    azure account import [path to .publishsettings file]


##Azure Configuration
Ok, we're ready to provision our cluster. We'll first need to create an affinity group for this cluster so the hosts selected for the CoreOS VMs are close to each other:

    azure account affinity-group create myapp-affinity -l "East US" -e "MyApp Affinity Group"

Next, create a cloud service for this cluster. We are going to assign containers to each of the hosts in this cluster to serve web traffic so we want to load balance incoming requests across them using a cloud service. This cloud service name needs to be unique across all of Azure, so choose a unique one:

    azure service create --affinity-group myapp-affinity myapp-cloud-service-name

Finally, we will create a virtual private network for our cluster to live inside.

    #TODO: this isnt working
    azure network vnet create --affinity-group myapp-affinity myapp-network

##Configure CoreOS cloud-config.yml file

The first thing we need to do is get a discovery token for etcd. 'etcd' is a distributed key-value store built on the Raft protocol and acts as a store for configuration information for CoreOS. Fleet, another part of the CoreOS puzzle, is a low-level init system built on 'etcd' that provides the functionality of Systemd over a distributed cluster.

This discovery token is configured in the cloud-init file called cloud-config.yml. This configures the CoreOS image once it is provisioned by Azure and, in particular, it injects the etcd discovery token into the virtual machine so that it knows which CoreOS cluster it belongs to. Its important to have a new and unique value for this, otherwise your cluster could fail to initialize correctly.

Let's provision a new one for our cluster:

    curl https://discovery.etcd.io/new

This will fetch a discovery URL that looks something like https://discovery.etcd.io/e6a84781d11952da545316cb90c9e9ab. Copy this and edit the cloud-config.yml files and paste this discovery token into it.

##Create Azure CoreOS VM Cluster

    azure vm create \
    --custom-data=cloud-config.yaml \
    --vm-size=Basic_A2 \
    --ssh=22 \
    --ssh-cert=../path/to/cert \
    --no-ssh-password \
    --vm-name=coreos1 \
    --virtual-network-name=myapp-network \
    --affinity-group=myapp-affinity \
    myapp-cloud-service-name \
    2b171e93f07c4903bcad35bda10acf22__CoreOS-Beta-494.0.0 \
    core

    azure vm create \
    --custom-data=cloud-config.yaml \
    --vm-size=Basic_A2 \
    --ssh=2222 \
    --ssh-cert=../path/to/cert \
    --no-ssh-password \
    --vm-name=coreos2 \
    --virtual-network-name=myapp-network \
    --affinity-group=myapp-affinity \
    myapp-cloud-service-name \
    2b171e93f07c4903bcad35bda10acf22__CoreOS-Beta-494.0.0 \
    core

##References
http://azure.microsoft.com/en-us/documentation/articles/xplat-cli/
https://coreos.com/docs/launching-containers/launching/fleet-using-the-client/
https://coreos.com/docs/running-coreos/cloud-providers/azure/
https://github.com/timfpark/coreos-azure