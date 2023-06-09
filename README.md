# Dapr on Azure Kubernetes Service (AKS)

This how-to guide combines [three Dapr docs articles](#resources) and explains how to run Dapr apps on Azure Kubernetes Service. You don't need to follow the three original articles, as this guide contains all the steps necessary.

In this guide:

- An AKS cluster will be created.
- Dapr will be installed on the cluster.
- Redis will be installed as the state store.
- Two Dapr applications will be deployed to the cluster.

![Dapr on AKS](images/dapr-on-k8s.png)

The NodeJS application has a `neworder` POST endpoint that persists order IDs, and an `order` GET endpoint to retrieve the latest order ID.

The Python application creates the order IDs and calls the `neworder` endpoint of the NodeJS service in a continuous loop.

## Prerequisites  

- [Azure account](https://azure.microsoft.com/free/search/) with rights to create resources
- [Docker](https://docs.docker.com/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://github.com/helm/helm#install)
- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest)
- [Dapr CLI](https://docs.dapr.io/getting-started/install-dapr-cli/)
- Optional: [VSCode](https://code.visualstudio.com/) with [REST client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client)

## 1. Create an AKS cluster

1. Login to Azure using the CLI:

    ```bash
    az login
    ```

2. Set the default subscription:

    ```bash
    az account set --subscription <subscription-id>
    ```

3. Create a resource group:

    ```bash
    az group create --name <resource-group-name> --location <location>
    ```

    _Example_

    ```bash
    az group create --name dapr-aks-rg --location westeurope
    ```

4. Create an AKS cluster:

    ```bash
    az aks create --resource-group <resource-group-name> --name <cluster-name> --node-count 2 --enable-addons http_application_routing --generate-ssh-keys
    ```

    _Example_

    ```bash
    az aks create --resource-group dapr-aks-rg --name dapr-aks --node-count 2 --enable-addons http_application_routing --generate-ssh-keys
    ```

5. Get the access credentials for the AKS cluster:

    ```bash
    az aks get-credentials --resource-group <resource-group-name> --name <cluster-name>
    ```

    _Example_

    ```bash
    az aks get-credentials --resource-group dapr-aks-rg --name dapr-aks
    ```

6. Verify the connection to the cluster:

    ```bash
    kubectl cluster-info
    ```

    Expected response:

    ```bash
    Kubernetes control plane is running at <CONTROL_PLANE_URL>
    addon-http-application-routing-nginx-ingress is running at <INGRESS_URL>:80 <INGRESS_URL>:443
    CoreDNS is running at <CONTROL_PLANE_URL>/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    Metrics-server is running at <CONTROL_PLANE_URL>/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
    ```

7. Verify the two nodes are deployed:

    ```bash
    kubectl get nodes
    ```

     Expected response:

    ```bash
    NAME                                STATUS   ROLES   AGE   VERSION
    aks-nodepool1-27324113-vmss000000   Ready    agent   20h   v1.24.9
    aks-nodepool1-27324113-vmss000001   Ready    agent   20h   v1.24.9
    ```

## 2. Setup Dapr on AKS

1. Initialize Dapr on the provisioned AKS cluster:

    ```bash
    dapr init --kubernetes --wait
    ```

    Expected response:

    ```bash
    Making the jump to hyperspace...
    Container images will be pulled from Docker Hub
    Deploying the Dapr control plane to your cluster...
    Success! Dapr has been installed to namespace dapr-system. To verify, run `dapr status -k' in your terminal.
    ```

2. Verify the Dapr services are running:

    ```bash
    dapr status -k
    ```

    Expected response:

    ```bash
    NAME                   NAMESPACE    HEALTHY  STATUS   REPLICAS  VERSION  AGE  CREATED
    dapr-sidecar-injector  dapr-system  True     Running  1         1.10.4   1m   2023-03-21 11:04.03
    dapr-sentry            dapr-system  True     Running  1         1.10.4   1m   2023-03-21 11:04.03
    dapr-operator          dapr-system  True     Running  1         1.10.4   1m   2023-03-21 11:04.03
    dapr-placement-server  dapr-system  True     Running  1         1.10.4   1m   2023-03-21 11:04.03
    dapr-dashboard         dapr-system  True     Running  1         0.12.0   1m   2023-03-21 11:04.03
    ```

    Ensure that all services are healthy and running before continuing.

## 3. Add a Redis State Store

1. Add a reference to the bitnami Redis Helm chart:

    ```bash
    helm repo add bitnami https://charts.bitnami.com/bitnami
    ```

    Expected response:

    ```bash
    "bitnami" has been added to your repositories
    ```

2. Update Helm:

    ```bash
    helm repo update
    ```

    Expected response:

    ```bash
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "bitnami" chart repository
    Update Complete. ⎈Happy Helming!⎈
    ```

3. Install Redis:

    ```bash
    helm install redis bitnami/redis --set image.tag=6.2
    ```

    Expected response:

    ```bash
    NAME: redis
    LAST DEPLOYED: Tue Mar 21 11:40:28 2023
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    CHART NAME: redis
    CHART VERSION: 17.8.7
    APP VERSION: 7.0.10

    ** Please be patient while the chart is being deployed **

    Redis can be accessed on the following DNS names from within your cluster:

    redis-master.default.svc.cluster.local for read/write operations (port 6379)
    redis-replicas.default.svc.cluster.local for read-only operations (port 6379)
    ...
    ```

4. Verify the Redis pods are running:

    ```bash
    kubectl get pods
    ```

    Expected response:

    ```bash
    NAME               READY   STATUS    RESTARTS   AGE
    redis-master-0     1/1     Running   0          13m
    redis-replicas-0   1/1     Running   0          13m
    redis-replicas-1   1/1     Running   0          12m
    redis-replicas-2   1/1     Running   0          11m
    ```

5. Ensure you are in the root directory of this repo and run:

    ```bash
    kubectl apply -f ./resources/redis.yaml
    ```

    Expected response:

    ```bash
    component.dapr.io/statestore created
    ```

## 4. Deploy the Node app

1. Ensure you're in the root of the repository and apply the Node app configuration to deploy the Node app:

    ```bash
    kubectl apply -f ./resources/node.yaml
    ```

    Expected response:

    ```bash
    service/nodeapp created
    deployment.apps/nodeapp created
    ```

2. Watch the status of the deployment:

    ```bash
    kubectl rollout status deploy/nodeapp
    ```

    This should eventually result in:

    ```bash
    deployment "nodeapp" successfully rolled out
    ```

3. Get the external IP of the service:

    ```bash
    kubectl get svc nodeapp
    ```

    Expected response:

    ```bash
    NAME      TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
    nodeapp   LoadBalancer   <INTERNAL_IP>   <EXTERNAL_IP>   80:31198/TCP   49m
    ```

4. Verify the service is running using curl or a REST client:

    ```bash
    curl http://<EXTERNAL_IP>/ports
    ```

    Expected response:

    ```json
    {"DAPR_HTTP_PORT":"3500","DAPR_GRPC_PORT":"50001"}
    ```

5. Submit an order to the app using curl or a REST client:

    ```bash
    curl --request POST --data "@sample.json" --header Content-Type:application/json http://<EXTERNAL_IP>/neworder
    ```

6. Confirm that the order ID is persisted using curl or a REST client:

    ```bash
    curl http://<EXTERNAL_IP>/order
    ```

    Expected response:

    ```json
    { "orderId": "42" }
    ```

## 5. Deploy the Python app

1. Ensure you're in the root of the repository and apply the Node app configuration to deploy the Python app:

    ```bash
    kubectl apply -f ./resources/python.yaml
    ```

    Expected response:

    ```bash
    deployment.apps/pythonapp created
    ```

2. Watch the status of the deployment:

    ```bash
    kubectl rollout status deploy/pythonapp
    ```

    This should eventually result in:

    ```bash
    deployment "pythonapp" successfully rolled out
    ```

3. Follow the logs of the Node.js app:

    ```bash
    kubectl logs --selector=app=node -c node -f
    ```

    Expected response:

    ```bash
    Got a new order! Order ID: 44
    Successfully persisted state for Order ID: 44
    Got a new order! Order ID: 45
    Successfully persisted state for Order ID: 45
    Got a new order! Order ID: 46
    Successfully persisted state for Order ID: 46
    ...
    ```

4. Confirm that the latest order ID is persisted using curl or a REST client:

    ```bash
    curl http://<EXTERNAL_IP>/order
    ```

## Dashboard

1. To access the Dapr dashboard, run the following command:

    ```bash
    dapr dashboard -k
    ```

    Expected response:

    ```bash
    Dapr dashboard found in namespace: dapr-system
    Dapr dashboard available at http://localhost:8080
    ```

2. Explore the dashboard to drill down into the applications, components, and services.

3. If you want manage multiple Dapr clusters, have fail-safe upgrades/downgrades, seamless root certificate rotations, and recommendations on your Dapr configuration, then [Diagrid Conductor](https://www.diagrid.io/conductor) would be an interesting option.

## 6. Clean-up

Follow these steps to remove all the apps, components and cloud resources created in this how-to guide.

1. Navigate to the resources folder and run this command to delete the services and state resources:

    ```bash
    kubectl delete -f .
    ```

    Expected response:

    ```bash
    service "nodeapp" deleted
    deployment.apps "nodeapp" deleted
    service "pythonapp" deleted
    deployment.apps "pythonapp" deleted
    component.dapr.io "statestore" deleted
    ```

2. Delete the AKS cluster:

    ```bash
    az aks delete --name <CLUSTER_NAME> --resource-group <RESOURCE_GROUP_NAME>
    ```

     _Example_

    ```bash
    az aks delete --name dapr-aks --resource-group dapr-aks-rg
    ```

## Resources

1. [Setup an Azure Kubernetes Service (AKS) cluster](https://docs.dapr.io/operations/hosting/kubernetes/cluster/setup-aks/).
2. [Tutorial: Configure state store and pub/sub message broker](https://docs.dapr.io/getting-started/tutorials/configure-state-pubsub/)
3. [Hello Kubernetes](https://github.com/dapr/quickstarts/tree/master/tutorials/hello-kubernetes)

## More information

Read more about hosting [Dapr on Kubernetes](https://docs.dapr.io/operations/hosting/kubernetes/) on the Dapr docs site.

Any questions or comments about this sample? Join the [Dapr discord](https://bit.ly/dapr-discord) and post a message the `#kubernetes` channel.
Have you made something with Dapr? Post a message in the `#show-and-tell` channel, we love to see your creations!