# Azure Kubernetes Services Tutorial

## Deployment Steps

### Pre-requisite

1. docker
2. kubectl
3. helm
4. Azure CLI
5. Contributor access to Azure subscription.

### Steps 

1. Login to Azure Portal and create a new resource group, named "rg-agw-aks-demo" in West US region.
2. Create new VNet with below details:-</br>
a) Name: vnet-agw-aks-demo </br>
b) Address Space: 192.168.0.0/22</br>
c) Subnet Name: aks-subnet-agw-aks-demo</br>
d) Subnet Address Space: 192.168.0.0/24</br>
e) Service Endpoint: Microsoft.AzureCosmosDB</br>
3. Once the Vnet is created, add another subnet with name: agw-subnet-agw-aks-demo</br>
4. Create a k8s cluster with the below details:- (Rest is default)</br>
a) Name: k8s-cluster-agw-aks-demo</br>
b) DNS Prefix: k8s-cluster-agw-aks-demo</br>
c) Kubernetes Cluster Version: 1.12.6</br>
d) Region: West US</br>
5. Networking - Advance</br>
a) Virtual Network: vnet-agw-aks-demo</br>
b) Cluster Subnet: aks-subnet-agw-aks-demo</br>
c) Kubernetes Service address: 198.166.0.0/26</br>
d) Kubernetes DNS Service IP: 198.166.0.10</br>
e) Docker Bridge Address: 172.17.0.1/16</br>
f) Disable Monitoring.</br>
6. Validate and Create the cluster.</br>
7. Create a new application gateway with below details</br>
a) Name: gateway-agw-aks-demo</br>
b) Tier: Standard V2</br>
c) Capacity Type: Manual</br>
d) Subnet: Choose agw-subnet-agw-aks-demo </br>
8. Make sure to create a public IP with DNS Name label.</br>
9. Pull k8s credentials to your local with the below command. Make sure you are logged into correct a/c with 'az login'
```bash
   az aks get-credentials --resource-group=rg-agw-aks-demo --name=k8s-cluster-agw-aks-demo
```
10. Add aad-pod-identity with the below command 
```bash
    kubectl create -f ~/AppDev/training/azure/Aks/deployment.yaml   
```
11. Create Identity in the same resource group as the AKS nodes (typically the resource group with a MC_ prefix string)
```bash 
    az identity create -g MC_rg-agw-aks-demo_k8s-cluster-agw-aks-demo_westus -n agw-aks-demo-user 
```
12. Find the principal, resource and client ID for this identity with the below command
```bash
    az identity show -g MC_rg-agw-aks-demo_k8s-cluster-agw-aks-demo_westus -n agw-aks-demo-user 
```
13. Assign this new identity Contributor access on the application gateway
```bash
az role assignment create --role Contributor --assignee <principal ID from the command above> --scope <Resource ID of Application Gateway>
```

14. Assign this new identity Reader access on the resource group that the application gateway belongs to
```bash
az role assignment create --role Reader --assignee <principal ID from the command above> --scope <Resource ID of Application Gateway Resource Group>
```

15. Add the application-gateway-kubernetes-ingress helm repo and perform a helm update
```bash
    helm init
    helm repo add application-gateway-kubernetes-ingress https://azure.github.io/application-gateway-kubernetes-ingress/helm/
    helm repo update
```

16. Edit helm-config.yaml and fill in the values 
<p>The <identity-resource-id> and <identity-client-id> are the properties of the Azure AD Identity you setup in the previous section. You can retrieve this information by running the following command:
```bash
    az identity show -g <resourcegroup> -n <identity-name> 
```
Where <resourcegroup> is the resource group in which the top level AKS cluster object, Application Gateway and Managed Identify are deployed.</p>

17. Install the helm chart with the command
```bash
    helm install -f helm-config.yaml application-gateway-kubernetes-ingress/ingress-azure
```

18. Create "ConfigMap" and "Secret" for cosmos DB connection
```bash
    kubectl create configmap cosmos-properties --from-literal=cosmos.uri=https://cosmos-fspt-dev1-dmz.documents.azure.com:443/ --from-literal=cosmos.db=cosmos-fspt-dev1-db1
    kubectl create secret generic cosmos-secret --from-literal=cosmos.key=2wbROjQh3vc4orEX6fXfekoXmv2XXfxJ0AIZxjGARbEo2WM5xgqnYK2qsShA1gGHuq60teQokPmqkhT7jbYqHg==
```

19. Deploy "k8s-welcome" service with the below command
```bash
    kubectl apply -f k8s-welcome-service.yaml
```

20. Deploy "cosmos-reference" service with the below command
```bash
    kubectl apply -f cosmos-reference.yaml
```

21. Deploy "ip-tracing" service with the below command
```bash
    kubectl apply -f ip-tracing-service.yaml
```
22. Deploy context-based routing rules
```bash
    kubectl apply -f agw-deployment.yaml
```

### Scaling pods

1. Manually scale pods
```bash
    kubectl scale --replicas=5 deployment/k8s-welcome
```

2. Use below command to deploy metric-server
```bash
    git clone https://github.com/kubernetes-incubator/metrics-server.git
    kubectl create -f metrics-server/deploy/1.8+/
```

3. Use the below command to set autoscaling criteria. In this case its, based on CPU threshold of 50%, min 3 and max 10 pods
```bash
    kubectl autoscale deployment k8s-welcome --cpu-percent=50 --min=3 --max=10
```

4. 

### Scaling Nodes

1. Use the below command to scale nodes (or from Portal)
```bash
    az aks scale --resource-group rg-agw-aks-demo --name k8s-cluster-agw-aks-demo --node-count 5
```

2. Node Autoscaling is in preview. Please refer https://docs.microsoft.com/en-us/azure/aks/cluster-autoscaler for more details.