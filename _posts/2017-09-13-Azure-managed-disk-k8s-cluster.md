---
layout: post
title: Deploying an Azure Kubernetes cluster with managed disks
---

Azure managed disks is finally on Kubernetes on Azure. Let's use ACS engine with Kubernetes 1.7 to try it out.

## Requirements

- Bash 
- Azure CLI 2.0 you can install it [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest), or use the Azure Cloud Shell in your browser (I am currently using `azure-cli (2.0.16)`)
- [jq](https://github.com/stedolan/jq/wiki/Installation) for parsing data from json responses on bash.
- Azure subscription where you are the owner or have a Service Principal with at least Contributor role to the subscription.
- SSH keys
- [VSCode](https://code.visualstudio.com/docs/setup/setup-overview) for file editing

## Setting up the environment

1. First of all, let's download the latest version of the `acs-engine` in your computer. At the moment of writing this article, I am using v0.6.0 from their [list of releases](https://github.com/Azure/acs-engine/releases). If you are using Mac OS X, you can type in the following commands from your terminal:

    ```bash
    mkdir managed-k8s-acs
    cd managed-k8s-acs
    curl -L https://github.com/Azure/acs-engine/releases/download/v0.6.0/acs-engine-v0.6.0-darwin-amd64.tar.gz | tar zx
    ```
    Just if you would like to check, the MD5 sum of the binary is the following: `b124d5ca90dcf5bdd0d9da5699ba776b`.

1. Get the json file example from my gist [repository](https://gist.github.com/brusMX/23077b2764f35cb20830e47c27a0c0d7#file-managed-disks-cluster-json):

    ```bash
    curl -O https://gist.githubusercontent.com/brusMX/23077b2764f35cb20830e47c27a0c0d7/raw/fd844532581016220d9a07d855f7e39ffb7cfc00/managed-disks-cluster.json
    ```

1. Generate a Service Principal for your deployment, you can use my [obtainAzure.sh](https://gist.github.com/brusMX/96538826d582388cb3b73a66a023b332) file to obtain it:

    ```bash
    curl -O https://gist.githubusercontent.com/brusMX/96538826d582388cb3b73a66a023b332/raw/f6fe0ec3ac1ddbbd0ede7e9d49a6eb49026af6a2/obtainAzureSP.sh
    ./obtainAzureSP.sh
    ```

    You will be using `AZURE_CLIENT_ID` and `AZURE_CLIENT_SECRET` in the next step, so keep them close. Also, remember that if you create a Service Principal for a subscription that was not the default one, you will have to run the following command:

    ```bash
    az account set -s <<CHOSEN SUBSCRIPTION ID>>
    ```

1. Open VSCode: `code managed-disk-cluster.json` and update the file accordingly.

    1. Make sure to use your own dns, update the value of `dnsPrefix` in line 10
    1. Paste the content of (`cat ~/.ssh/id_rsa.pub`) your public ssh key into `keydata` in line 29.
    1. Put the value of `AZURE_CLIENT_ID` on `clientId` in line 35
    1. Update the value of `secret` with `AZURE_CLIENT_SECRET`

## Deploying the cluster

Now that we have all our files in order, we can proceed to use ACS-Engine to create the artifacts of our cluster and deploy them.

1. Create artifacts with ACS-Engine, you will see a folder called `_output` after you run the following command:

    ```bash
    ./acs-engine-v0.6.0-darwin-amd64/acs-engine generate managed-disks-cluster.json
    ```

1. Set up environment variables that will define where the cluster will be located:

    1. Choose a location:

        ```bash
        export LOC=southcentralus
        ```

    1. Name your Resource Group

        ```bash
        export RG_NAME=cluster-mngd-k8s-brusmx
        ```

    1. Obtain the dns name from your json file

        ```bash
        export RG_DNS_NAME=`cat managed-disks-cluster.json | jq -r ."properties"."masterProfile"."dnsPrefix"`
        ```

1. Create your resource group where everything will be deployed (if something goes bad, you can always delete the whole RG and start again)

    ```bash
    az group create --name "${RG_NAME}" --location $LOC
    ```

1. Deploy your cluster to Azure with the CLI

    ```bash
    az group deployment create --resource-group="${RG_NAME}" --template-file="_output/${RG_DNS_NAME}/azuredeploy.json" --name="${RG_DNS_NAME}" --parameters @_output/${RG_DNS_NAME}/azuredeploy.parameters.json
    ```

## Connect to your cluster and test it out

After the cluster has been succesfully deployed we can obtain our kubeconfig file and start interacting with it.

1. Obtain `.kubeconfig` file

    ```bash
    scp -o StrictHostKeyChecking=no azureuser@$RG_DNS_NAME.$LOC.cloudapp.azure.com:/home/azureuser/.kube/config config-$RG_DNS_NAME
    ```

1. Paste it in your home. For this case I will create an extra file so you don't have to worry about your previous k8s config files.

    ```bash
    mkdir ~/.kube
    mv config-$RG_DNS_NAME ~/.kube
    export  KUBECONFIG=~/.kube/config-$RG_DNS_NAME
    ```

1. Confirm that your cluster is up. You can start running the following commands and make sure that everything is working the way it should be:

    ```bash
    kubectl cluster-info

    Kubernetes master is running at https://k8scluster.southcentralus.cloudapp.azure.com
    Heapster is running at https://k8scluster.southcentralus.cloudapp.azure.com/api/v1/namespaces/kube-system/services/heapster/proxy
    KubeDNS is running at https://k8scluster.southcentralus.cloudapp.azure.com/api/v1/namespaces/kube-system/services/kube-dns/proxy
    kubernetes-dashboard is running at https://k8scluster.southcentralus.cloudapp.azure.com/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy
    tiller-deploy is running at https://k8scluster.southcentralus.cloudapp.azure.com/api/v1/namespaces/kube-system/services/tiller-deploy/prox
    ```

    ```bash
    kubectl get nodes

    NAME                    STATUS    AGE       VERSION
    k8s-agent-25202797-0    Ready     44s       v1.7.5
    k8s-master-25202797-0   Ready     52s       v1.7.5
    ```
