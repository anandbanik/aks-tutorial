# Azure Kubernetes Services

## Kubernetes

### About Kubernetes
Kubernetes (k8s) is an open-source system for automating deployment, scaling, and management of containerized applications. It groups containers that make up an application into logical units for easy management and discovery.

### Features
1. Start, stop, update, and manage a cluster of machines running containers in a consistent and maintainable way.
2. Particularly suited for horizontally scaleable, stateless, or 'microservices' application architectures.
3. Additional functionality to make containers easier to use in a cluster (reachability and discovery).
4. Kubernetes does NOT and will not expose all of the 'features' of the docker command line.

### Components

1. <b>Master</b>:- It typically consists of </br>
a) kube-apiserver</br>
b) kube-scheduler</br>
c) kube-controller-manager</br>
d) etcd</br>
e) kube-proxy</br>

2. <b>Node</b>:- It typically consists of </br>
a) kubelet</br>
b) kube-proxy</br>
c) cAdvisor</br>

3. <b>Pods</b>:-</br>
a) Single schedulable unit of work which neither move nor spans across machines.</br>
b) Usually one or more container</br>
c) Metadata about the container(s)</br>
d) Env vars – configuration for the container</br>
e) Every pod gets an unique IP by the container engine</br>

![Pods](images/Pods.PNG)

4. <b>Deployment</b>:-</br>
A Deployment controller provides declarative updates for Pods and ReplicaSets. You describe a desired state in a Deployment object, and the Deployment controller changes the actual state to the desired state at a controlled rate. You can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments.

5. <b>Services</b>:-</br>
a) How 'stuff' finds pods which could be anywhere</br>
b) Define:
    <ul>
    <li> What port in the container</li>
    <li> Labels on pods which should respond to this type of request</li>
    </ul>
c) Can define:
    <ul>
    <li> What the 'internal' IP should be</li>
    <li> What the 'external' IP should be</li>
    <li> What port the service should listen on</li>
    </ul>

![Services](images/Services.PNG)

## Deployment Steps

### Pre-requisite

1. Docker
2. kubectl
3. Helm
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

