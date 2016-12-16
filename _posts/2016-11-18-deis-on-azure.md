---
layout: post
title: Set up DEIS on Azure 
---

First, we will deploy an [Azure Container Service (ACS)](https://azure.microsoft.com/en-us/services/container-service/) running [Kubernetes](http://kubernetes.io/) with [ACS Engine](https://github.com/Azure/acs-engine). Then we will install [DEIS](https://deis.com/) on top of this cluster.

## Requirements

- [Windows 10 bash](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide) or other unix based terminal.
- [Go](https://golang.org/doc/install) programming language
- [jq](https://stedolan.github.io/jq/) to parse the json responses in the console
- [ACS Engine](https://github.com/Azure/acs-engine/blob/master/docs/acsengine.md#linux)
- [Kubectl](http://kubernetes.io/docs/getting-started-guides/kubectl/)
- [DEIS](https://deis.com/docs/workflow/quickstart/install-cli-tools/)
- [HELM](https://github.com/kubernetes/helm/blob/master/docs/install.md)

## Steps

1. The easiest way to do this is by getting the new Azure CLI installed. You can follow the instructions [here](https://docs.microsoft.com/en-us/cli/azure/install-az-cli2), and basically if you are running Bash on Windows 10 (Build 14362+) with Python 2.7.6, you should do the following:

    ```Bash
    sudo apt-get update
    sudo apt-get install -y libssl-dev libffi-dev
    sudo apt-get install -y python-dev
    curl -L https://aka.ms/InstallAzureCli | sudo bash
    ```

    Or if you are running Mac OS X:

    ```bash
    curl -L https://aka.ms/InstallAzureCli | sudo bash
    ```
1. Login to your Azure account:

    ```bash
    az login
    ```
    This will provide us the information that we need to use in the following commands. You will obtain something like this:

    ```json
    {
        "cloudName": "AzureCloud",
        "id": "98xxx7d-bxx8-sxx2-txx4-a2dxxxxxxxp0",
        "isDefault": true,
        "name": "My awesome subscription",
        "state": "Enabled",
        "tenantId": "xxxxxxx-xxx-xxx-xxx-fxxxxx4",
        "user": {
        "name": "super@cool.mail.com",
        "type": "user"
        }
    },
    ```

1. Use the subscription id `"id": "98xxx7d-bxx8-sxx2-txx4-a2dxxxxxxxp0"`from the previous step and replace it in the following commands to set it as a shell variable that can be used in the creation of the [RBAC role](https://docs.microsoft.com/en-us/azure/active-directory/role-based-access-control-what-is). We will collect this info in the `SP_JSON` environment variable to use it later.

    ```bash
    SUBSCRIPTION_ID=98xxx7d-bxx8-sxx2-txx4-a2dxxxxxxxp0
    az account set --subscription="${SUBSCRIPTION_ID}"
    SP_JSON=`az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/${SUBSCRIPTION_ID}"`
    ```

    This has set up the json response in the environment variable `SP_JSON` we can run `echo $SP_JSON` and it will output something like this:

    ```json
    {
        "appId": "47xxxxx-xxxx-xxxx-xxxx-xxxxxdac2",
        "name": "http://azure-cli-2016-12-xxxxx-47",
        "password": "faxxxx-xxxx-xxxx-xxxx-xxxx5bc222",
        "tenant": "72fxxxx-xxxx-xxxx-xxxx-xxxxx11db47"
    }
    ```

    For convenience we can use `jq` to set up more precise environment variables from the json we just collected, these variables will be used in the following steps instead of actually having to remember the actual value:

    ```bash
    SP_NAME=`echo $SP_JSON | jq -r '.name'`
    SP_PASS=`echo $SP_JSON | jq -r '.password'`
    SP_TENANT=`echo $SP_JSON | jq -r '.tenant'`
    ```

    You can verify that your Service Principal works by logging in with the following command.

    ```bash
    az login --service-principal -u "${SP_NAME}" -p "${SP_PASS}" --tenant "${SP_TENANT}"
    ```

    You should obtain the same info you already had when you first logged in:

    ```json
    {
        "cloudName": "AzureCloud",
        "id": "98xxx7d-bxx8-sxx2-txx4-a2dxxxxxxxp0",
        "isDefault": true,
        "name": "My awesome subscription",
        "state": "Enabled",
        "tenantId": "xxxxxxx-xxx-xxx-xxx-fxxxxx4",
        "user": {
            "name": "http://azure-cli-2016-12-xxxxx-47",
            "type": "servicePrincipal"
        }
    },
    ```

1. Create the resource group

    ```bash
    RG_NAME=myresourcegroup
    az resource group create --name "${RG_NAME}" --location southcentralus
    ```

    You should get the following output:

    ```json
    {
        "id": "/subscriptions/6xxxxxxx-xxxx-xxxx-xxx-4xxxxxxxx/resourceGroups/<"RG_NAME">",
        "location": "southcentralus",
        "managedBy": null,
        "name": <"RG_NAME">,
        "properties": {
            "provisioningState": "Succeeded"
        },
        "tags": null
    }
    ```

1. Since Go and ACS Engine is already installed. We will proceed to generate a template from the `example/kubernetes.json` file in the `acs-engine` folder that you have previously installed.

    ```bash
    cd $GOPATH/src/github.com/Azure/acs-engine
    cp examples/kubernetes.json examples/my_kubernetes.json
    ```

    Now lets open the file:

    ```json
    {
        "apiVersion": "vlabs",
        "properties": {
            "orchestratorProfile": {
            "orchestratorType": "Kubernetes"
            },
            "masterProfile": {
            "count": 1,
            "dnsPrefix": "<<< YOUR CLUSTER NAME >>>",
            "vmSize": "Standard_D2_v2"
            },
            "agentPoolProfiles": [
                {
                    "name": "agentpool1",
                    "count": 3,
                    "vmSize": "Standard_D2_v2",
                    "availabilityProfile": "AvailabilitySet"
                }
            ],
            "linuxProfile": {
            "adminUsername": "azureuser",
            "ssh": {
                "publicKeys": [
                    {
                        "keyData":"<<< YOUR SSH PUBLIC KEY >>>"
                    }
                ]
            }
            },
            "servicePrincipalProfile": {
                "servicePrincipalClientID": "<<< YOUR SP NAME >>>",
                "servicePrincipalClientSecret": "<<< YOUR SP PASSWORD >>>"
            }
        }
    }
    ```

    Replace the `<<< YOUR CLUSTER NAME >>>`, `<<< YOUR SSH PUBLIC KEY >>>` `<<< YOUR SP NAME >>>` and `<<< YOUR SP PASSWORD >>>` with your information.

1. Generate the json files with the ACS engine

    ```bash
    ./acs-engine examples/my_kubernetes.json
    ```

    Deploy them:

    ```bash
    CLUSTER_NAME=mysupercluster
    cd _output/<INSTANCE>
    az resource group deployment create --resource-group="{RG_NAME}" --template-file="azuredeploy.json" --name="${CLUSTER_NAME}" --parameters @azuredeploy.parameters.json
    ```

1. Copy the `.kube/config` file into your console. If you have more kubernetes clusters you can create a custom name for your file and then use the environment variable `$KUBECONFIG` to indicate the one you want to use. If this is your only cluster, just make sure to have it the config file in this location `~/.kube/config` using the folowing command:

    ```bash
    scp azureuser@myclustername.northeurope.cloudapp.azure.com:.kube/config ~/.kube/config
    ```

1. Check the cluster info now:

    ```bash
    kubectl cluster-info
    ```

1. Install DEIS, run each command one by one:

    ```bash
    helmc repo add deis https://github.com/deis/charts
    helmc fetch deis/workflow-v2.8.0
    helmc generate -x manifests workflow-v2.8.0
    helmc install workflow-v2.8.0
    ```
1. See how yor pods are being created:

    ```bash
    kubectl --namespace=deis get pods -w
    ```

    Your output should look like this:

    ```bash
    NAME                                     READY     STATUS    RESTARTS   AGE
    deis-builder-415246326-pngpz             0/1       Running   0          2m
    deis-controller-1590631305-hykp4         0/1       Running   3          2m
    deis-database-510315365-mrbk5            1/1       Running   0          2m
    deis-logger-9212198-tp03l                1/1       Running   3          2m
    deis-logger-fluentd-162yx                1/1       Running   0          1m
    deis-logger-fluentd-i8hrp                1/1       Running   0          1m
    deis-logger-fluentd-obrfr                1/1       Running   0          1m
    deis-logger-fluentd-ys3oo                1/1       Running   0          1m
    deis-logger-redis-663064164-6em9a        1/1       Running   0          2m
    deis-minio-3160338312-ugsi3              1/1       Running   0          2m
    deis-monitor-grafana-432364990-dyqzl     1/1       Running   0          2m
    deis-monitor-influxdb-2729526471-iypdy   1/1       Running   0          1m
    deis-monitor-telegraf-47d6l              1/1       Running   0          1m
    deis-monitor-telegraf-ehy32              1/1       Running   0          1m
    deis-monitor-telegraf-ljts0              1/1       Running   0          1m
    deis-monitor-telegraf-xx1cz              1/1       Running   0          1m
    deis-nsqd-3264449345-i2ojb               1/1       Running   0          1m
    deis-registry-2182132043-7vmjd           1/1       Running   1          1m
    deis-registry-proxy-hraa8                1/1       Running   0          1m
    deis-registry-proxy-ow6vu                1/1       Running   0          1m
    deis-registry-proxy-vyrfx                1/1       Running   0          1m
    deis-registry-proxy-w6yfs                1/1       Running   0          1m
    deis-router-2457652422-fu7du             1/1       Running   0          1m
    deis-workflow-manager-2210821749-t884r   1/1       Running   0          1m
    NAME                               READY     STATUS    RESTARTS   AGE
    deis-controller-1590631305-hykp4   1/1       Running   3          2m
    deis-builder-415246326-pngpz   1/1       Running   0         2m
    deis-monitor-telegraf-47d6l   0/1       Error     0         2m
    deis-monitor-telegraf-47d6l   1/1       Running   1         2m
    deis-monitor-telegraf-ehy32   0/1       Error     0         2m
    deis-monitor-telegraf-ehy32   1/1       Running   1         2m
    ```

1. Get the ip from the router

    ```bash
    kubectl --namespace=deis describe svc deis-router
    ```

    You should look for `LoadBalancer Ingeress` IP address.

1. Using that ip use the controller url to register a new user in the cluster

    ```bash
    deis register http://deis."${LB_IP}".nip.io
    ```

1. Create an empty application:

    ```bash
    ~ ❯❯❯ deis create --no-remote
    Creating Application... done, created sanest-radiator
    If you want to add a git remote for this app later, use `deis git:remote -a sanest-radiator`
    ```