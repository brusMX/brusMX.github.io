---
layout: post
title: Deploying Hybrid Kubernetes cluster (Windows and Linux) on Azure
---

Windows and Linux agent pools working together on Azure.
We will deploy a Kubernetes cluster with 2 nodes on a Windows agent pool and 2 nodes on a Linux Pool, CNI enabled, Kubernetes v1.10, HyperV containers on Windows, Open Service Broker enabled.

Optional, a KeyVault to store Service Principals and RBAC support with AAD.

## Requirements

- Bash. Ubuntu bash for Windows 10 or your bash compatible terminal on Mac OS X or Linux (zsh, tsh)
- Azure CLI 2.0 you can install it [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest), or use the Azure Cloud Shell in your browser (I am currently using `azure-cli (2.0.23)`)
- (Optional) [jq](https://github.com/stedolan/jq/wiki/Installation) for parsing data from json responses on bash.
- Azure subscription where you are the owner or have a Service Principal with at least Contributor role to the subscription.
- SSH keys in your local machine
- [VSCode](https://code.visualstudio.com/docs/setup/setup-overview) for file edition.

## Setting up the environment


1. Download the latest version of the `acs-engine` in your computer. At the moment of writing this article, I am using v0.14.6 from their [list of releases](https://github.com/Azure/acs-engine/releases/latest). If you are using Ubuntu Bash, you can type in the following commands from your terminal:

    ```bash
    mkdir hybrid-k8s
    cd hybrid-k8s
    curl -L https://github.com/Azure/acs-engine/releases/download/v0.14.6/acs-engine-v0.14.6-linux-amd64.tar.gz | tar zx
    cp acs-engine-v0.14.6-linux-amd64/acs-engine .
    chmod +x acs-engine
    rm -rf acs-engine-v0.14.6-linux-amd64/
    ```
    Just if you would like to check, the MD5 sum of the binary is the following: `8de3cb98f93238287f8c41c3d3c0dc1f`.

1. Make sure you have the latest version (`az --version`) of the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-linux?view=azure-cli-latest#install-or-update), I have in this moment `azure-cli (2.0.30)`. And make sure you are logged in (`az login`) to the right subscription (`az account list -o table`).

1. Generate a Service Principal for your deployment, you can use my [getAzureServicePrincipal.sh](https://gist.github.com/brusMX/bab2224dc1b0c26d3aef4799cb97c045) file to obtain it. You can run the following commands in your bash terminal to get one:

    ```bash
    curl -O https://gist.githubusercontent.com/brusMX/bab2224dc1b0c26d3aef4799cb97c045/raw/bf05884b5aeca9ae0c455af3ce0e695ec372cccc/getAzureServicePrincipal.sh
    chmod +x getAzureServicePrincipal.sh
    ./getAzureServicePrincipal.sh
    ```

    You will be using `AZURE_CLIENT_ID` and `AZURE_CLIENT_SECRET` in the next step, so keep them close. Also, remember that if you create a Service Principal for a subscription that was not the default one, you will have to run the following command to change into the desired subscription:

    ```bash
    az account set -s <<CHOSEN SUBSCRIPTION ID>>
    ```

1. Create a `.env` file up environment variables that will define most of the configurations needed for your cluster. Just create on your `hybrid-k8s` folder a file named `.env` and edit it with VSCode:

    ```bash
    export LOC=southcentralus
    export RG_NAME=hybridk8srg001
    export DNS_NAME=hybrid-cluster-001
    # Copy the Service Principal you obtained from the ./getAzureServicePrincipal.sh script here
    export AZURE_CLIENT_ID=XXXXXXXX-XXXXXX-XXXX-XXXX-XXXXXXXXX
    export AZURE_CLIENT_NAME=http://azure-cli-XXXXXXXXXXX
    export AZURE_CLIENT_SECRET=XXXXXXXX-XXXXXX-XXXX-XXXX-XXXXXXXXX
    export AZURE_TENANT_ID=XXXXXXXX-XXXXXX-XXXX-XXXX-XXXXXXXXX
    ```

    And source it in your terminal:

    ```bash
    source .env
    ```

1. Create your resource group where everything will be deployed (if something goes bad, you can always delete the whole RG and start again)

    ```bash
    az group create --name "${RG_NAME}" --location $LOC
    ```
1. You need an SSH Key without a passphrase, you are able to see if you have `cat $HOME/.ssh/id_rsa.pub`. If you are not sure you can create one ssh key by runninf the following command:

    ```bash
    ssh-keygen -t rsa -b 4096 -N ""
    ```

    You can cat it again and make sure it is there `cat $HOME/.ssh/id_rsa.pub`.

1. Create the json file that will be used to deploy the cluster. If you have all the right environment variables set up, you can copy paste the following command on bash:

    ```bash
    cat <<EOT >> hybrid-cluster.json
    {
        "apiVersion": "vlabs",
        "properties": {
            "orchestratorProfile": {
                "orchestratorType": "Kubernetes",
                "orchestratorRelease": "1.10",
                "kubernetesConfig": {
                "enableRbac": true,
                "networkPolicy": "azure",
                "enableAggregatedAPIs": true
                }
            },
            "masterProfile": {
            "count": 1,
            "dnsPrefix": "`echo $DNS_NAME`",
                "vmSize": "Standard_D2_v2"
            },
            "agentPoolProfiles": [
                {
                "name": "linuxpool1",
                "count": 1,
                "vmSize": "Standard_DS2_v2",
                "storageProfile" : "ManagedDisks",
                "availabilityProfile": "AvailabilitySet"
                },
                {
                "name": "windowspool1",
                "count": 2,
                "vmSize": "Standard_D2s_v3",
                "storageProfile" : "ManagedDisks",
                "availabilityProfile": "AvailabilitySet",
                "osType": "Windows"
                }
            ],
            "linuxProfile": {
            "adminUsername": "`echo $USERNAME`",
            "ssh": {
                "publicKeys": [
                {
                    "keyData": "`cat $HOME/.ssh/id_rsa.pub`"
                }
                ]
            }
            },
            "servicePrincipalProfile": {
            "clientId": "`echo $AZURE_CLIENT_ID`",
            "secret": "`echo $AZURE_CLIENT_SECRET`"
            }
        }
    }
    EOT
    ```

1. Cat the file `cat hybrid-cluster.json` and make sure that the variables **dnsPrefix**, **adminUsername**, **keyData**, **clientId** and **secret** were populated. If not open the file with VSCode and edit it with the right info: `code hybrid-cluster.json` and update the file accordingly.

    1. Make sure to use an unique name for `dnsPrefix`.
    1. Paste the content of your public SSH key into `keydata`.
    1. Put the value of `AZURE_CLIENT_ID` on `clientId`.
    1. Update the value of `secret` with `AZURE_CLIENT_SECRET`

## Deploying the cluster

Now that we have all our files in order, we can proceed to use ACS-Engine to create the artifacts of our cluster and deploy them.

1. Create artifacts with ACS-Engine, you will see a folder called `_output` after you run the following command:

    ```bash
    ./acs-engine generate hybrid-cluster.json
    ```

1. Deploy your cluster to Azure with the ACS Engine binary

    ```bash
    az group deployment create --resource-group="${RG_NAME}" --template-file="_output/${DNS_NAME}/azuredeploy.json" --name="${DNS_NAME}" --parameters @_output/${DNS_NAME}/azuredeploy.parameters.json
    ```

## Connect to your cluster and test it out

After the cluster has been succesfully deployed we can obtain our kubeconfig file and start interacting with it.

1. Obtain `.kubeconfig` file

    ```bash
    scp -o StrictHostKeyChecking=no $DNS_NAME.$LOC.cloudapp.azure.com:/home/$USERNAME/.kube/config config-$DNS_NAME
    ```

1. Paste this file in your the. For this case I will create an extra file so you don't have to worry about your previous k8s config files.

    ```bash
    mkdir ~/.kube
    mv config-$RG_DNS_NAME ~/.kube
    export  KUBECONFIG=~/.kube/config-$DNS_NAME
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
    k8s-agent-25202797-0    Ready     44s       v1.9
    k8s-master-25202797-0   Ready     52s       v1.9
    ```
