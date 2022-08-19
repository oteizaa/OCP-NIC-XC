# OCP-NIC-XC

Ensure to have the Azure CLI 2.6.0 or later

  az --version

# Register the resource providers

1) If you have multiple Azure subscriptions, specify the relevant subscription ID:

   ```bash
   az account set --subscription <SUBSCRIPTION ID>
   ```

2) Register the Microsoft.RedHatOpenShift resource provider:
  az provider register -n Microsoft.RedHatOpenShift --wait

# 3) Register the Microsoft.Compute resource provider:
az provider register -n Microsoft.Compute --wait

# 4) Register the Microsoft.Storage resource provider:
az provider register -n Microsoft.Storage --wait

# 5) Register the Microsoft.Authorization resource provider:
az provider register -n Microsoft.Authorization --wait

# 6) ARO preview features are available on a self-service, opt-in basis. Preview features are provided "as is" and "as available," and they are excluded from the service-level agreements and limited warranty. Preview features are partially covered by customer support on a best-effort basis. As such, these features are not meant for production use.
az feature register --namespace Microsoft.RedHatOpenShift --name preview

# Create a virtual network containing two empty subnets

# 1) Set the following variables in the shell environment in which you will execute the az commands.
LOCATION=eastus                 # the location of your cluster
RESOURCEGROUP=aro-rg            # the name of the resource group where you want to create your cluster
CLUSTER=cluster                 # the name of your cluster

# 2) Create a resource group.
az group create \
  --name $RESOURCEGROUP \
  --location $LOCATION

# 3) Create a virtual network.
az network vnet create \
   --resource-group $RESOURCEGROUP \
   --name aro-vnet \
   --address-prefixes 10.0.0.0/22

# 4) Add an empty subnet for the master nodes.
az network vnet subnet create \
  --resource-group $RESOURCEGROUP \
  --vnet-name aro-vnet \
  --name master-subnet \
  --address-prefixes 10.0.0.0/23 \
  --service-endpoints Microsoft.ContainerRegistry

# 5) Add an empty subnet for the worker nodes.
az network vnet subnet create \
  --resource-group $RESOURCEGROUP \
  --vnet-name aro-vnet \
  --name worker-subnet \
  --address-prefixes 10.0.2.0/23 \
  --service-endpoints Microsoft.ContainerRegistry

# 6) Disable subnet private endpoint policies on the master subnet. This is required for the service to be able to connect to and manage the cluster.
az network vnet subnet update \
  --name master-subnet \
  --resource-group $RESOURCEGROUP \
  --vnet-name aro-vnet \
  --disable-private-link-service-network-policies true

# Create the cluster
az aro create \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --vnet aro-vnet \
  --master-subnet master-subnet \
  --worker-subnet worker-subnet

