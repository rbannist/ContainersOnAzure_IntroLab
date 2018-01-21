# Intro to Containers on Azure
This lab aims to show a few ways you can quickly deploy container workloads to Azure. 
<br><br>
# What is it?

This intro lab serves to guide you on a few ways you can deploy a container on Azure, namely:

*	Deploy a container on App Service PaaS platform
*	Deploy a container on an Azure Container Instance (managed Kubernetes instance)
*	Deploy a managed Kubernetes cluster on Azure using Azure Kubernetes Service (AKS) and deploy our container onto it
* Write to Azure Cosmos DB. [Cosmos DB](https://azure.microsoft.com/en-us/services/cosmos-db/) is Microsoft's globally distributed, multi-model database 
* Use [Application Insights](https://azure.microsoft.com/en-us/services/application-insights/) to track custom events in the container
* Deploy Helm to your Kubernetes cluster
<br><br>
# Technology used

* Our container contains a swagger enabled API developed in Go which writes a simple order via json to your specified Cosmos DB and tracks custom events via Application Insights.
<br><br>
# Preparing for this lab

For this Lab you will require:

* Install the Azure CLI 2.0, get it here - https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
* Install Docker, get it here - https://docs.docker.com/engine/installation/
* Install Kubectl, get it here - https://kubernetes.io/docs/tasks/tools/install-kubectl/
* Install Postman, get it here - https://www.getpostman.com - this is optional but useful

When using the Azure CLI, after logging in, if you have more than one subscripton you may need to set the default subscription you wish to perform actions against. To do this use the following command:

```
az account set --subscription "<your requried subscription guid>"
```
<br><br>
## 1. Provisioning a Cosmos DB instance

Let's start by creating a Cosmos DB instance in the portal or using CLI, this is a quick process.

### Portal:
Navigate to the Azure portal and create a new Azure Cosmos DB instance, enter the following parameters:

* ID: <yourdbinstance>
* API: Select MongoDB as the API as our container API will use this driver
* ResourceGroup: <yourresourcegroup>
* Location: westeurope

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/CosmosDB.png)


Once the DB is provisioned, we need to get the Database Username and Password, these may be found in the Settings --> Connection Strings section of your DB. We will need these to run our container, so copy them for convenient access. See below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/DBKeys.png)


### CLI:

```
az group create -n <yourresourcegroup> -l westeurope
```

```
az cosmosdb create -n <yourcosmosdbid> -g <yourresourcegroup> --kind MongoDB
```

```
az cosmosdb list-connection-strings -n <yourcosmosdbid> -g <yourresourcegroup>
```
-->  Store the output in a text file for later use.
<br><br>
If you wish to see the keys in isolation:

```
az cosmosdb list-keys -n <yourcosmosdbid> -g <yourresourcegroup>
```
<br><br>
## 2. Provisioning an Application Insights instance

In the Azure portal, select create new Application Insights instance, enter the following parameters:

* Name: <yourappinsightsinstance>
* Application Type: General
* ResourceGroup: <yourresourcegroup>
* Location: westeurope

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/ApplicationInsights.png)

Once Application Insights is provisioned, we need to get the Instrumentation key, this may be found in the Overview section. We will need this to run our container, so copy it for convenient access. See below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/AppKey.png)
<br><br>
## 3. Provisioning an Azure Container Registry instance

If you would like an example of how to setup an [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/) instance via ARM, have a look [here](https://github.com/shanepeckham/CADScenario_Recommendations)

### Portal:

Navigate to the Azure Portal and select create new Azure Container Registry, enter the following parameters:

* Registry Name: <yourcontainerregistryinstance>
* ResourceGroup: <yourresourcegroup>
* Location: westeurope
* Admin User: Enable
* SKU: Basic or Classic
* Storage Account: Select the default value provided (if Classic)

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/ContainerRegistry.png)


### CLI:

```
az acr create -n <youracrname> -g <yourresourcegroup> --sku Basic --admin-enabled true -l westeurope
```

```
az ad sp create-for-rbac --scopes /subscriptions/<yoursubsriptionid>/resourceGroups/<yourresourcegroup>/providers/Microsoft.ContainerRegistry/registries/<youracrname> --role Owner --password <yourpassword>
```

```
az acr show -n <youracrname> --query loginServer
```

-->  Save the ouput in the text file for later use.
<br><br>
## 4. Pull the container to your environment and set the environment keys

Open up your docker machine in Guacamole and type the following:

```
docker pull shanepeckham/go_order_sb
```

We will now test the image locally to ensure that it is working and connecting to our CosmosDB and Application Insights instances. The keys you copied for the DB and App Insights keys are set as environment variables within the container, so we will need to ensure we populate these.

The environment keys that need to be set are as follows:
* DATABASE: <your cosmodb username from step 1>
* PASSWORD: <your cosmodb password from step 1>
* INSIGHTSKEY: <you app insights key from step 2>
* SOURCE: This is a free text field which we will use specify where we are running the container from. The values 'Localhost', 'AppService', 'ACI' and 'K8' are applicable to our tests.

So, to run the container on your local machine, enter the following command, substituting your environment variable values:

```
sudo docker run --name go_order_sb -p 8080:8080 --network=host -e DATABASE="<your cosmodb username from step 1>" -e PASSWORD="<your cosmodb password from step 1>" -e INSIGHTSKEY="<you app insights key from step 2>" -e SOURCE="localhost"  -i -t shanepeckham/go_order_sb
```
Note, the application runs on port 8080 which we will bind to the host as well (-p switch).  However, we're running the container directly on the 'host network' as well in this example (i.e. on 172.31.0.0/24) in order to access it from the neighbouring windows machine(s).

If you wish to explore docker host's networks enter the following commands

```
sudo docker network ls

sudo docker network inspect <bridgenetworkid> | <hostnetworkid>
```

If all goes well, you should see the application running on <dockervmipaddress>:8080.  'ifconfig -a' on the Docker VM will give you the IP address that you need.  See below for an example:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/localrun.png)

Now you can navigate to <dockervmipaddress>:8080/swagger and test the api. Select the 'POST' /order/ section, select the button "Try it out" and enter some values in the json provided and select "Execute", see below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/swagger.png)

If the request succeeded, you will get a CosmosDB Id returned for the order you have just placed, see below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/swaggerresponse.png)


You can also test using CURL on the Mgmt/Windows VM.  Firsty, install CURL:

```
choco install curl -y
```

Then send a POST call:

```
curl -X POST "http://<dockervmipaddress>:8080/v1/order/" -H  "accept: application/json" -H  "content-type: application/json" -d "{  \"EmailAddress\": \"<emailaddress<>\",  \"ID\": \"string\",  \"PreferredLanguage\": \"ENU\",  \"Product\": \"Latte\",  \"Source\": \"Localhost\",  \"Total\": 1}"
```
<br><br>
We can now go and query CosmosDB to check our entry there, in the Azure portal, navigate back to your Cosmos DB instance and go to the section Data Explorer. We can now query for the order(s) that we placed. A collection called 'orders' will have been created within your database, you can then apply a filter for the id we created, namely:

```
{"id":"<orderid>"}
```

See below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/CosmosQuery.png)
<br><br>
## 5. Retag the image and upload it your private Azure Container Registry

Navigate to the Azure Container Registry instance you provisioned within the Azure portal. Click on the *Quick Start* blade, this will provide you with the relevant commands to upload a container image to your registry, see below:

![alt text](https://github.com/shanepeckham/CADScenario_Recommendations/blob/master/images/quicksstartacs.png)

Now we will push the image up to the Azure Container Registry, enter the following (from the quickstart screen):

``` 
docker login <yourcontainerregistryinstance>.azurecr.io
```

To get the username and password, navigate to the *Access Keys* blade, see below:

![alt text](https://github.com/shanepeckham/CADScenario_Recommendations/blob/master/images/acskeys.png)

You will receive a 'Login Succeeded' message. Now type the following:
```
docker tag shanepeckham/go_order_sb <yourcontainerregistryinstance>.azurecr.io/go_order_sb
docker push <yourcontainerregistryinstance>.azurecr.io/go_order_sb
```
Once this has completed, you will be able to see your container uploaded to the Container Registry within the portal, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/registryrepo.png)
<br><br>
## 6. Deploy the container to App Services

We will now deploy the container to Azure App Services via the Azure CLI. If you would like an example of how to setup an [App Service Application](https://docs.microsoft.com/en-us/azure/app-service-web/app-service-linux-intro) instance via ARM and associate the container with your Azure Container Registry, have a look [here](https://github.com/shanepeckham/CADScenario_Recommendations)

[Login to your Azure subscription via the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli) and enter the following first command to create your App service plan:

```
az appservice plan create -g <yourresourcegroup> -n <yourappserviceplan> --is-linux
```

Upon receiving the 'provisioningState': 'Succeeded' json response, enter the following to create your app which will run our API:

```
az webapp create -n <youruniquewebappname> -p <yourappserviceplan> -g <yourresourcegroup> --deployment-container-image-name <yourcontainerregistryinstance>.azurecr.io/go_order_sb
```


Upon receiving the successful completion json response, we will now associate our container from our private Azure Registry to the App Service App, type the following):

```
az webapp config container set -n <youruniquewebappname> -g <yourresourcegroup> --docker-custom-image-name <yourcontainerregistryinstance>.azurecr.io/go_order_sb:latest --docker-registry-server-url https://<yourcontainerregistryinstance>.azurecr.io --docker-registry-server-user <your acr admin username> --docker-registry-server-password <your acr admin password>
```

### Associate the environment variables with API App

Now we need to go and set the environment variables for our container to ensure that we can connect to our Cosmos DB and Application Insights. Navigate to the *Application Settings* pane within the Azure portal for your Web App and add the following entries in the 'App Settings' section, namely:

The environment keys that need to be set are as follows:
* DATABASE: <your cosmodb username from step 1>
* PASSWORD: <your cosmodb password from step 1>
* INSIGHTSKEY: <you app insights key from step 2>
* SOURCE: This is a free text field which we will use specify where we are running the container from.
* WEBSITES_PORT: 8080 

See below:
![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/AppSettingsWeb.png)

Or, when using Azure CLI:

```
az webapp config appsettings set -n <youruniquewebappname> -g <yourresourcegroup> --settings "DATABASE=<your cosmodb username from step 1>" "PASSWORD=<your cosmodb password from step 1>" "INSIGHTSKEY=<you app insights key from step 2>" "SOURCE=AppServices" "WEBSITES_PORT=8080"
```

Now we can test our app service container, the URL should be https://<youruniquewebappname>.azurewebsites.net/swagger but you can also navigate to the Overview section to get the URL for your API, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/App_URI.png)

Ensure you add ```/swagger``` on to the end of the URL to access the Swagger API test harness.


### Stream the logs from the App Service container

To see the log stream of your container running in the web app, navigate to: ```https://<youruniquewebappname>.scm.azurewebsites.net/api/logstream```
<br><br>
## 7. Deploy the container to Azure Container Instance

Now we will deploy our container to [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/). 

In the command terminal, login using the AZ CLI and we will start off by creating a new resource group for our Container instance. At the time of writing this functionality is still in preview and is thus not available in all regions (it is currently available in westeurope, eastus, westus), hence why we will create a new resource group just in case.

Enter the following:

```
az group create --name <yourACIresourcegroup> --location <westeurope, eastus, westus>
```

### Associate the environment variables with Azure Container Instance

We will now deploy our container instance via an ARM template, which is [here](https://github.com/rbannist/ContainersOnAzure_MiniLab/blob/master/azuredeploy.json) but before we do, we need to edit this document to ensure we set our environment variables.


In the document, the following section needs to be amended, adding your environment keys like you did before:

```

"properties": {
                "containers": [
                    {
                        "name": "[variables('container1name')]",
                        "properties": {
                            "image": "[variables('container1image')]",
                            "environmentVariables": [
                                {
                                    "name": "DATABASE",
                                    "value": "<your cosmodb username from step 1>"
                                },
                                {
                                    "name": "PASSWORD",
                                    "value": "<your cosmodb password from step 1>"
                                },
                                {
                                    "name": "INSIGHTSKEY",
                                    "value": "<you app insights key from step 2>"
                                },
                                {
                                    "name": "SOURCE",
                                    "value": "ACI"
                                }
                            ],

```


Once this document is saved, we can create the deployment via the az CLI. Enter the following:

```
az group deployment create --name <yourACIname> --resource-group <yourACIresourcegroup> --template-file /<path to your file>/azuredeploy.json
```

It is also possible to create the container instance via the Azure CLI directly.

```
az container create -n go-order-sb -g <yourACIresourcegroup> -e DATABASE=<your cosmodb username from step 1> PASSWORD=<your cosmodb password from step 1> INSIGHTSKEY=<your app insights key from step 2> SOURCE="ACI"--image <yourcontainerregistryinstance>.azurecr.io/go_order_sb:latest --registry-password <your acr admin password>
```

You can check the status of the deployment by issuing the container list command:

```
az container show -n go-order-sb -g <yourACIresourcegroup> -o table
```

Once the container has moved to "Succeeded" state you will see your external IP address under the "IP:ports" column, copy this value and navigate to http://yourACIExternalIP:8080/swagger and test your API like before.
<br><br>
## 8. Deploy the container to an Azure Kubernetes Service cluster

Here we will deploy an Azure Kubernetes Service (AKS) cluster which is a managed Kubernetes environment in Azure.  We will then run the container on the cluster.  Please use the Cloud Shell due to a bug in Azure-CLI version installed on your Windows VM.

We will start by once again creating a resource group for our cluster.  In a command window enter the following:

```
az group create --name <yourresourcegroupk8> --location westeurope
```

az aks create -n arc-we-coa-k8s01 -g arc-we-coa-rg02

az aks get-credentials --resource-group arc-we-coa-rg02 --name arc-we-coa-k8s01

kubectl get nodes

touch goordersb.yaml

nano goordersb.yaml

apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: goordersb
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: goordersb
    spec:
      containers:
      - name: goordersb
        image: arcwecoaacr01.azurecr.io/go_order_sb
        ports:
        - containerPort: 8080
          name: goordersb
---
apiVersion: v1
kind: Service
metadata:
  name: goordersb
spec:
  type: LoadBalancer
  ports:
  - port: 8080
  selector:
    app: goordersb
---


kubectl create secret docker-registry arcwecoaacr01.azurecr.io --docker-server=arcwecoaacr01.azurecr.io --docker-username=arcwecoaacr01 --docker-password=CUI32Wd=lbr+8D5nIbflfCop73ZZX+b6 --docker-email=barichar@microsoft.com

kubectl create -f goordersb.yaml

kubectl get service goordersb --watch
--> Wait for 'EXTERNAL-IP'




Upon receiving your "provisioningState": "Succeeded" json response, enter the following:

```
az acs create --orchestrator-type kubernetes --resource-group <yourresourcegroupk8> --name <yourk8cluster> --generate-ssh-keys
```
In case you have not already, install the kubernetes client:

```
=======
az acs kubernetes install-cli

```

You will now be able to connect to your cluster with the following command:

```
az acs kubernetes get-credentials --resource-group=<yourresourcegroupk8> --name=<yourk8cluster>
```

And to access your Kubernetes graphical dashboard enter:

```
az acs kubernetes browse -g <yourresourcegroupk8> -n <yourk8cluster> 
```

Note, it is always a good idea to apply an auto shutdown policy to your VMs to avoid unnecessary costs for a test cluster, you can do this in the portal by navigating to the VMs provisioned within your resource group <yourresourcegroupk8> and navigating to the Auto Shutdown section for each one, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/autoshutdown.png)

### Register our Azure Container Registry within Kubernetes

We now want to register our private Azure Container Registry with our Kubernetes cluster to ensure that we can pull images from it. Enter the following within your command window:

```
kubectl create secret docker-registry <yourcontainerregistryinstance>.azurecr.io --docker-server=<yourcontainerregistryinstance>.azurecr.io --docker-username=<youracradminusername> --docker-password=<youracradminpassword> --docker-email=<youremailaddress>
```

In the Kubernetes dashboard you should now see this created within the secrets section:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/K8secrets.png)



### Associate the environment variables with container we want to deploy to Kubernetes

We will now deploy our container via a yaml file, which is [here](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/go_order_sb.yaml) but before we do, we need to edit this file to ensure we set our environment variables and ensure that you have set your private Azure Container Registry correctly:

```

spec:
      containers:
      - name: goordersb
        image: <containerregistry>.azurecr.io/go_order_sb
        env:
        - name: DATABASE
          value: "<your cosmodb username from step 1>""
        - name: PASSWORD
          value: "<your cosmodb password from step 1>""
        - name: INSIGHTSKEY
          value: ""<you app insights key from step 2>""
        - name: SOURCE
          value: "K8"
        ports:
        - containerPort: 8080
      imagePullSecrets:
        - name: <yourcontainerregistry>
```

Once the yaml file has been updated, we can now deploy our container. Within the command line enter the following:

```
kubectl create -f ./<your path>/go_order_sb.yaml
```
You should get a success message that a deployment and service has been created. Navigate back to the Kubernetes dashboard and you should see the following:

#### Your deployments running 

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/k8deployments.png)

#### Your three pods

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/k8pods.png)

#### Your service and external endpoint

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/k8service.png)

You can now navigate to http://k8serviceendpoint:8080/swagger and test your API
<br><br>
## 8. Deploy the container to an Azure Container Engine and manage it from within your Kubernetes cluster

Now we will deploy our container to Azure Container Instances and use the [ACI connector](https://github.com/azure/aci-connector-k8s) to manage it from within our Kubernetes cluster.

### Create a Service Principle

A service principal is required to allow the ACI Connector to create resources in your Azure subscription. You can create one using the az CLI using the instructions below.

Find your ``` subscriptionId ``` with the az CLI:

```
$ az account list -o table
Name                                             CloudName    SubscriptionId                        State    IsDefault
-----------------------------------------------  -----------  ------------------------------------  -------  -----------
Pay-As-You-Go                                    AzureCloud   12345678-9012-3456-7890-123456789012  Enabled  True
```

Use ``` az ``` to create a Service Principal that can perform operations on your resource group:
```
$ az ad sp create-for-rbac --role=Contributor --scopes /subscriptions/<subscriptionId>/resourceGroups/<yourresourcegroupk8>
```
After one or a few attempts, you should see the following json structure being output:
```
{
  "appId": "<redacted>",
  "displayName": "azure-cli-2017-07-19-19-13-19",
  "name": "http://azure-cli-2017-07-19-19-13-19",
  "password": "<redacted>",
  "tenant": "<redacted>"
}
```

#### Install the ACI Connector

Edit the [aci_connector_go_order_sb.yaml](https://github.com/rbannist/ContainersOnAzure_MiniLab/blob/master/aci_connector_go_order_sb.yaml) and input environment variables using the values above:

* AZURE_CLIENT_ID: insert appId
* AZURE_CLIENT_KEY: insert password
* AZURE_TENANT_ID: insert tenant
* AZURE_SUBSCRIPTION_ID: insert subscriptionId

```
$ kubectl create -f ./<your_path>/aci-connector.yaml 
deployment "aci-connector" created

$ kubectl get nodes -w
NAME                        STATUS                     AGE       VERSION
aci-connector               Ready                      3s        1.6.6
k8s-agentpool1-31868821-0   Ready                      5d        v1.7.0
k8s-agentpool1-31868821-1   Ready                      5d        v1.7.0
k8s-agentpool1-31868821-2   Ready                      5d        v1.7.0
k8s-master-31868821-0       Ready,SchedulingDisabled   5d        v1.7.0
```

You should now the see the ACI Connector running within your Kubernetes cluster, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/k8acsconnector.png)

### Deploy the container to Azure Container Instance managed by Kubernetes and set environment variables

We will now deploy our container via a yaml file again, which is [here](https://github.com/rbannist/ContainersOnAzure_MiniLab/blob/master/go_order_sb_aci_node.yaml) but before we do, we need to edit this file to ensure we set our environment variables.

Now we want to add the environment variables and ensure that you have set your private Azure Container Registry correctly:
```
spec:
  containers:
  - name: goordersb
    image: <yourcontainerregistry>.azurecr.io/go_order_sb
    env:
    - name: DATABASE
      value: ""
    - name: PASSWORD
      value: ""
    - name: INSIGHTSKEY
      value: ""
    - name: SOURCE
      value: "K8ACI"
    ports:
      - containerPort: 8080
  imagePullSecrets:
    - name: <yourcontainerregistry>
  dnsPolicy: ClusterFirst
  nodeName: aci-connector
  
  ```
  
Deploy our container using the following command:
```
kubectl create -f ./<your_path>/go_order_sb_aci_node.yaml
```
Once deployed you should now see your container instances running, one within your cluster, and one running on the ACI Connector pod, see below:
  
 ![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/K8acipod.png)

Click on the ACI Connector pod, mark down the IP address, and navigate to the following URL to test your API:
```
http://<your_ACI_Connector_pod_IP_address>:8080/swagger
```

 ### Deploy Helm to your Kubernetes cluster
 Firstly, download [Helm](https://github.com/kubernetes/helm/releases/tag/v2.5.1), unpack it and place it within your PATH, or ammend your path environment variable to include the location of the helm binary.

 Initialise the helm configuration with

 ```
 helm init
 ```

 Use Helm to search for and install stable/traefik, and ingress controller to enable inbound requests for your builds.

 ```
 $ helm search traefik
NAME            VERSION DESCRIPTION
stable/traefik  1.3.0   A Traefik based Kubernetes ingress controller w...

$ helm install stable/traefik --name ingress
```

Once the ingress controller has been deployed, check the IP address that has been allocated within the Pod:

```
kubectl get svc -w
NAME              CLUSTER-IP   EXTERNAL-IP     PORT(S)                      AGE
ingress-traefik   10.0.98.22   23.101.66.197   80:31765/TCP,443:31391/TCP   10h
kubernetes        10.0.0.1     <none>          443/TCP                      13h
```


### View container telemetry in Application Insights

The container we have deployed writes simple events to Application Insights with a time stamp but we could write much richer metrics. Application Insights provides a number of prebuilt dashboards to view application statistics alongside a query tool for getting deep custom insights. For the purposes of this intro we will simply expose the custom events we have tracked, namely the commit to the Azure CosmosDB.

In portal navigate to the Application Insights instance you provisioned and 'Metrics Explorer', see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/MetricsExplorer.png)
 
Click edit on one of the charts, select a TimeRange and set the Filters to 'Custom Event'. This will retrieve all of the writes to CosmosDB, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/Filter.png)

Now we can Search the events by the source, for example 'K8' to retrieve only Kubernetes cluster writes, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/Search.png)

Finally, for more powerful queries, select the 'Analytics' button, see below:

![alt text](https://github.com/shanepeckham/ContainersOnAzure_MiniLab/blob/master/images/Analytics.png)