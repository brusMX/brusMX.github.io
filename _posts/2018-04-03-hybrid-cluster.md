---
layout: post
title: Deploying Hybrid Kubernetes cluster (Windows and Linux) on Azure
---

Windows and Linux agent pools working together on Azure.
We will deploy a Kubernetes cluster with 2 nodes on a Windows agent pool and 2 nodes on a Linux Pool and Kubernetes v1.9.

Optional, a KeyVault to store Service Principals and RBAC support with AAD.

## Requirements

- Bash. **Windows Users** the easiest way is to [install Ubuntu bash for Windows 10](https://www.howtogeek.com/249966/how-to-install-and-use-the-linux-bash-shell-on-windows-10/) or your bash compatible terminal on Mac OS X or Linux (zsh, tsh)
- Azure CLI 2.0 you can install it [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest), or use the Azure Cloud Shell in your browser (I am currently using `azure-cli (2.0.30)`)
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
    # Get the Subscription ID you will get it from the previous command
    export AZURE_SUB_ID=XXXXXXX-XXXX
    # Copy the Service Principal you obtained from the ./getAzureServicePrincipal.sh script here
    export AZURE_CLIENT_ID=XXXXXXXX-XXXXXX-XXXX-XXXX-XXXXXXXXX
    export AZURE_CLIENT_NAME=http://azure-cli-XXXXXXXXXXX
    export AZURE_CLIENT_SECRET=XXXXXXXX-XXXXXX-XXXX-XXXX-XXXXXXXXX
    export AZURE_TENANT_ID=XXXXXXXX-XXXXXX-XXXX-XXXX-XXXXXXXXX
    # Usernames and pass for the vms, make sure to update them
    export VM_USER=azureuser1
    export WIN_VM_PASS=W!n*Us3r01
    ```

    And source it in your terminal:

    ```bash
    source .env
    ```

1. Create your resource group where everything will be deployed (if something goes bad, you can always delete the whole RG and start again)

    ```bash
    az group create --name $RG_NAME --location $LOC
    ```
    If you are getting weird errors about the RG_NAME on windows, just install dos2unix `apt-get install dos2unix` and convert the special chars with 

    ```bash
    dos2unix .env
    ```

1. You need an SSH Key without a passphrase, you are able to see if you have one already with the following command: `cat $HOME/.ssh/id_rsa.pub`. If you don't have one, you can easily create one ssh key by runnin the following command:

    **NOTE: This command will ask you to override a previous SSH key, in this case is better to create one with a custom name right next to it. Just provide a new name when prompted**

    ```bash
    ssh-keygen -t rsa -b 4096 -N ""
    ```

    You can now obtain it with this command:

    ```bash
    cat $HOME/.ssh/id_rsa.pub
    ```

1. Create the json file that will be used to deploy the cluster. If you have all the right environment variables set up, you can copy paste the following command on your bash terminal and press `enter`. This will auto populate the file with your variables:

    ```bash
    cat <<EOT > hybrid-cluster.json
    {
        "apiVersion": "vlabs",
        "properties": {
            "orchestratorProfile": {
                "orchestratorType": "Kubernetes",
                "orchestratorRelease": "1.9"
            },
            "masterProfile": {
                "count": 3,
                "dnsPrefix": "`echo $DNS_NAME`",
                "vmSize": "Standard_DS2_v2"
            },
            "agentPoolProfiles": [
                {
                    "name": "linuxpool1",
                    "count": 2,
                    "vmSize": "Standard_DS2_v2",
                    "storageProfile" : "ManagedDisks",
                    "availabilityProfile": "AvailabilitySet"
                },
                {
                    "name": "windowspool1",
                    "count": 2,
                    "vmSize": "Standard_D2_v3",
                    "availabilityProfile": "AvailabilitySet",
                    "osType": "Windows"
                }
            ],
            "windowsProfile": {
                "adminUsername": "`echo $VM_USER`",
                "adminPassword": "`echo $WIN_VM_PASS`"
            },
            "linuxProfile": {
                "adminUsername": "`echo $VM_USER`",
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
    ```

    Finally type in CAPS `EOT` and press `Enter` on your keyboard to close and save the file.

1. Cat the file `cat hybrid-cluster.json` and make sure that the variables **dnsPrefix**, **adminUsername**, **keyData**, **clientId** and **secret** were populated. If not, you can edit `.env` and re-run the command above and it will replace the `hybrid-cluster.json` file with the new variables. Alternatively, you can open the file with VSCode and edit it with the right info: `code hybrid-cluster.json`.

    1. Make sure to use an unique name for `dnsPrefix`.
    1. Paste the content of your public SSH key into `keydata`, **Remember, you need a key without a passphrase**.
    1. Put the value of `AZURE_CLIENT_ID` from your service principal on `clientId`.
    1. Update the value of `secret` with `AZURE_CLIENT_SECRET` from that same Service Principal.

## Deploying the cluster

Now that we have all our files in order, we can proceed to use ACS-Engine to create the artifacts of our cluster and deploy them.

1. Create artifacts with ACS-Engine, you will see a folder called `_output` after you run the following command:

    ```bash
    ./acs-engine generate hybrid-cluster.json
    ```

1. Deploy your cluster to Azure with the ACS Engine binary. This will ask you to login again in your subscription

    ```bash
    ./acs-engine deploy _output/$DNS_NAME/apimodel.json -l $LOC --subscription-id $AZURE_SUB_ID -g $RG_NAME
    ```
    You will see something like the following:

    ```bash
    WARN[0001] To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code B2PEZ5NUM to authenticate.
    INFO[0168] Starting ARM Deployment (hybridk8rg01-2035783392). This will take some time...
    ```
    Alternatively, you can deploy it using the Azure CLI:

    ```bash
    az group deployment create --resource-group="${RG_NAME}" --template-file="_output/${DNS_NAME}/azuredeploy.json" --name="${DNS_NAME}" --parameters @_output/${DNS_NAME}/azuredeploy.parameters.json
    ```

## Connect to your cluster and test it out

After the cluster has been succesfully deployed we can obtain our kubeconfig file and start interacting with it.

1. Obtain `.kubeconfig` file

    ```bash
        scp -o StrictHostKeyChecking=no $VM_USER@$DNS_NAME.$LOC.cloudapp.azure.com:/home/$VM_USER/.kube/config config-$DNS_NAME
    ```

1. Paste this file in your the. For this case I will create an extra file so you don't have to worry about your previous k8s config files.

    ```bash
    mkdir ~/.kube
    mv config-$DNS_NAME ~/.kube
    export  KUBECONFIG=~/.kube/config-$DNS_NAME
    ```

1. If you don't have `kubectl` installed. You can run from your bash with Azure Cli the following command:

    ```bash
    sudo az aks install-cli
    ```

    You should see an output like this:

    ```bash
    Downloading client to /usr/local/bin/kubectl from https://storage.googleapis.com/kubernetes-release/release/v1.10.0/bin/linux/amd64/kubectl

    Please ensure that /usr/local/bin is in your search PATH, so the `kubectl` command can be found.
    ```

1. Confirm that your cluster is up. You can start running the following commands and make sure that everything is working the way it should be:

    ```bash
    kubectl cluster-info

    Kubernetes master is running at https://hybridk8s004.southcentralus.cloudapp.azure.com
    Heapster is running at https://hybridk8s004.southcentralus.cloudapp.azure.com/api/v1/namespaces/kube-system/services/heapster/proxy
    KubeDNS is running at https://hybridk8s004.southcentralus.cloudapp.azure.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    kubernetes-dashboard is running at https://hybridk8s004.southcentralus.cloudapp.azure.com/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
    Metrics-server is running at https://hybridk8s004.southcentralus.cloudapp.azure.com/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
    tiller-deploy is running at https://hybridk8s004.southcentralus.cloudapp.azure.com/api/v1/namespaces/kube-system/services/tiller-deploy:tiller/proxy
    ```

    ```bash
    kubectl get nodes -o wide

    NAME                        STATUS    ROLES     AGE       VERSION   EXTERNAL-IP   OS-IMAGE                    KERNEL-VERSION   CONTAINER-RUNTIME
    18466k8s9010                Ready     <none>    9m        v1.9.6    <none>        Windows Server Datacenter   10.0.16299.309
                                docker://17.6.2
    18466k8s9011                Ready     <none>    9m        v1.9.6    <none>    Windows Server Datacenter   10.0.16299.309
                                docker://17.6.2
    k8s-linuxpool1-18466771-0   Ready     agent     15m       v1.9.6    <none>    Debian GNU/Linux 9 (stretch)   4.13.0-1011-azure   docker://1.13.1
    k8s-master-18466771-0       Ready     master    15m       v1.9.6    <none>    Debian GNU/Linux 9 (stretch)   4.13.0-1011-azure   docker://1.13.1
    ```

1. Test the windows containers. Create a `windows-service.yaml` with the following content:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
        name: win-webserver
        labels:
            app: win-webserver
    spec:
        ports:
            # the port that this service should serve on
        - port: 80
          targetPort: 80
        selector:
            app: win-webserver
        type: LoadBalancer
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
        labels:
            app: win-webserver
        name: win-webserver
    spec:
        replicas: 1
        template:
            metadata:
                labels:
                    app: win-webserver
                name: win-webserver
            spec:
                containers:
                - name: windowswebserver
                  image: microsoft/windowsservercore:1709
                  command:
                  - powershell.exe
                  - -command
                  - "<#code used from https://gist.github.com/wagnerandrade/5424431#> ; $$listener = New-Object System.Net.HttpListener ; $$listener.Prefixes.Add('http://*:80/') ; $$listener.Start() ; $$callerCounts = @{} ; Write-Host('Listening at http://*:80/') ; while ($$listener.IsListening) { ;$$context = $$listener.GetContext() ;$$requestUrl = $$context.Request.Url ;$$clientIP = $$context.Request.RemoteEndPoint.Address ;$$response = $$context.Response ;Write-Host '' ;Write-Host('> {0}' -f $$requestUrl) ;  ;$$count = 1 ;$$k=$$callerCounts.Get_Item($$clientIP) ;if ($$k -ne $$null) { $$count += $$k } ;$$callerCounts.Set_Item($$clientIP, $$count) ;$$header='<html><body><H1>Windows Container Web Server</H1>' ;$$callerCountsString='' ;$$callerCounts.Keys | % { $$callerCountsString+='<p>IP {0} callerCount {1} ' -f $$_,$$callerCounts.Item($$_) } ;$$footer='</body></html>' ;$$content='{0}{1}{2}' -f $$header,$$callerCountsString,$$footer ;Write-Output $$content ;$$buffer = [System.Text.Encoding]::UTF8.GetBytes($$content) ;$$response.ContentLength64 = $$buffer.Length ;$$response.OutputStream.Write($$buffer, 0, $$buffer.Length) ;$$response.Close() ;$$responseStatus = $$response.StatusCode ;Write-Host('< {0}' -f $$responseStatus)  } ; "
                nodeSelector:
                    beta.kubernetes.io/os: windows
    ```

    And deploy it with:

    ```bash
    kubectl apply -f windows-service.yaml

    service "win-webserver" created
    deployment.extensions "win-webserver" created
    ```

    In a few minutes you will see the service up and running and the containers created:

    ```bash
    kubectl get all

    NAME                             READY     STATUS              RESTARTS   AGE
    win-webserver-6f5c5c676d-nhvd9   0/1       ContainerCreating   0          1m

    NAME            TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
    kubernetes      ClusterIP      10.0.0.1      <none>        443/TCP        39m
    win-webserver   LoadBalancer   10.0.121.17   <pending>     80:32326/TCP   1m

    NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    win-webserver   1         1         1            0           1m

    NAME                       DESIRED   CURRENT   READY     AGE
    win-webserver-6f5c5c676d   1         1         0         1m

    NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    win-webserver   1         1         1            0           1m

    NAME                       DESIRED   CURRENT   READY     AGE
    win-webserver-6f5c5c676d   1         1         0         1m
    ```

    It might take up to 10 minutes to get an external IP. But when you have it, you can use it in your browser and you will see something like this:

    ```html
    <h1>Windows Container Web Server</h1>

    <p>IP 10.240.0.5 callerCount 1</p>
    ```


1. On the other hand, you can deploy a simple linux nginx service with the following command:

    ```bash
    kubectl run my-nginx --image=nginx --replicas=2 --port=80
    kubectl expose deployment my-nginx --port=80 --type=LoadBalancer
    ```

    Wait for the IP to come back with `kubectl get svc` and browse into it. You should see the welcome page from NGINX running on Linux

### References

- [How to get logs from Windows Agents](https://github.com/andyzhangx/Demo/tree/master/debug#q-how-to-get-k8s-kubelet-logs-on-windows-agent)


