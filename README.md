# Azure Managed Application and AKS with Managed Identity

This repo includes some sample commands and ARM templates for experimenting with Azure Managed Application that deploys an AKS resource and Managed Identities. See related videos at [Azure Managed Application with AKS and deployment-time or cross-tenant role assignments to VM and Pod Managed Identities](https://medium.com/@ArsenVlad/azure-managed-application-with-aks-and-deployment-time-or-cross-tenant-role-assignments-to-vm-and-3ebce7d607c2)

> Important: Currently (August 2021), there are some capability-gaps when Azure Managed Application deploys an AKS resource. Before developing Azure Managed Application that includes AKS or containers, please review the document "[Usage of Azure Kubernetes Services (AKS) and containers in managed application](https://docs.microsoft.com/en-us/azure/marketplace/plan-azure-app-managed-app#usage-of-azure-kubernetes-service-aks-and-containers-in-managed-application)" which lists important rules and limitations.

> When building your Azure Application ARM templates for submission to Azure Marketplace, please make sure to carefully follow all of the guidelines and practices described [here](https://github.com/Azure/azure-quickstart-templates/blob/master/1-CONTRIBUTION-GUIDE/best-practices.md) and be ready to make fixes and changes based on [manual review feedback](https://docs.microsoft.com/en-us/azure/marketplace/partner-center-portal/azure-apps-review-feedback).

## Deployment-time User Assigned Identity to AKS Node Resource Group

A way to add User Assigned Identity role assignments **at deployment time** to the AKS Node Resource Group

1. Create user assigned managed identity during ARM template deployment
1. Explicitly define a name for the AKS nodeResourceGroup instead of having AKS create the name automatically (i.e. MC_xxx_yyy_region) so that we can use this name for role assignment
1. Use a nested resourceGroup-scoped inline deployment to create Owner (or Contributor depending on what permissions are needed) role assignment for the user assigned identity at the scope of the AKS Node Resource Group. This must be a nested deployment since scope must align with the deployment.
1. Now, we can use this user assigned identity in our bootstraping VMs (or Deployment Scripts) to make changes to all node pool VMSSs that make up the AKS cluster (remember to do it for new node pools that maybe created later to ensure they also have the needed identity)

See [ama-aks folder](./ama-aks) for an example template managed app with AKS and nested role assignment templates.

> Publisher must have "Owner" access

```bash
# On bootstrapping VM, login using its user assigned identity
az login --identity -u /subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avama2-mrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/avama2mi1

# Assign identity to the VMSS
az vmss identity assign -g <VMSS_RESOURCE_GROUP> -n <VMSS_NAME> --identities /subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avama2-mrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/avama2mi1
```

## Publisher get AKS credentials

After deploying AKS with a managed identity (from customer subscription), get the AKS credentials using publisher identity.

```bash
az aks get-credentials -g avama2-mrg -n avama2
```

## Using AKS node pool's VMSS Identity

### Start Alpine Linux Pod and install curl and jq

```bash
kubectl run --rm -it test --image=alpine
apk add curl jq
```

### Try accessing Instance Metadata from within the Pod

```bash
# Get instance metadata
curl -H Metadata:true -s --noproxy "*" "http://169.254.169.254/metadata/instance?api-version=2020-06-01" | jq

# Trying to get system assigned managed identity from the pod will fail since VMSS does not have a System Assigned identity
curl -H Metadata:true -s --noproxy "*" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fmanagement.azure.com%2F"

# Get specific user assigned managed identity assigned to the VMSS
curl -H Metadata:true -s --noproxy "*" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fmanagement.azure.com%2F&mi_res_id=/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/MC_avama2-mrg_avama2_westus/providers/Microsoft.ManagedIdentity/userAssignedIdentities/avama2-agentpool" | jq
```

### Access storage using Bearer token

Prior to executing commands below, we need assign "Storage Blob Reader" role to the avama2-agentpool managed identity to the relevant storage account (avadlsgen2 as in the example below) or the resource group containing the storage account. We can perform this assignment at deployment-time by improving the template in [ama-aks folder](./ama-aks) or after deployment using publisher credentials and the managedIdentityRoleAssignment.json template as described below.

```bash
# Get access token for accessing storage for the specific user assigned identity of the VMSS
export ACCESS_TOKEN=$(curl -H Metadata:true -s --noproxy "*" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://storage.azure.com/&mi_res_id=/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/MC_avama2-mrg_avama2_westus/providers/Microsoft.ManagedIdentity/userAssignedIdentities/avama2-agentpool" | jq -r '.access_token')

# List files and directories
curl -H "Authorization: Bearer $ACCESS_TOKEN" "https://avadlsgen2.dfs.core.windows.net/folder1?recursive=false&resource=filesystem" | jq

# Read file
curl -H "Authorization: Bearer $ACCESS_TOKEN" https://avadlsgen2.dfs.core.windows.net/folder1/t1.csv
```

## Pod Specific Identity

See <https://github.com/Azure/aad-pod-identity/> for more details.

### Deploy aad-pod-identity

```bash
# Pod Identity with Kubenet is not recommended and used here (via the nmi.allowedNetworkPluginKubenet=true) just as an example
# See <https://azure.github.io/aad-pod-identity/docs/configure/aad_pod_identity_on_kubenet/>
helm repo add aad-pod-identity https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts --set nmi.allowNetworkPluginKubenet=true
helm install aad-pod-identity aad-pod-identity/aad-pod-identity
```

### Create Managed Identity and Reader Role Assignment

> Publisher must have "Owner" access

```bash
# Create user assigned identity in the managed resource group
az identity create -n avama2mi1 -g avama2-mrg

# Get principalId of the created managed identity
az identity show -n avama2mi1 -g avama2-mrg -o json

# Get principalId of the AKS managed identity <https://github.com/Azure/aad-pod-identity/blob/master/docs/readmes/README.role-assignment.md>
az aks show -g avama2-mrg -n avama2aks --query identityProfile.kubeletidentity.objectId -otsv

# Publisher (if has "Owner" access) must use ARM template since need to set delegatedManagedIdentityResourceId which is not yet exposed via Azure CLI

# Assign "Virtual Machine Contributor" to AKS node resource group that contains the VMSS of the nodes
az deployment group create -g avama2-aks-rg --template-file managedIdentityRoleAssignment.json --parameters principalId="5d534332-2b97-4ad5-8cf2-7f70ed91226b" principalType="ServicePrincipal" roleDefinitionId="/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/providers/Microsoft.Authorization/roleDefinitions/d73bb868-a0df-4d4d-bd69-98a00b01fccb" scope="/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avama2-aks-rg" delegatedManagedIdentityResourceId="/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourcegroups/avama2-mrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/avama2mi1"

# "Managed Identity Operator" to managed resource group that will contain the managed identities that will be assigned to pods
az deployment group create -g avama2-mrg --template-file managedIdentityRoleAssignment.json --parameters principalId="5d534332-2b97-4ad5-8cf2-7f70ed91226b" principalType="ServicePrincipal" roleDefinitionId="/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/providers/Microsoft.Authorization/roleDefinitions/f1a07417-d97a-45cb-824c-7a7467783830" scope="/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avama2-mrg" delegatedManagedIdentityResourceId="/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avama2-mrg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/avama2mi1"
```

### Create AzureIdentity and AzureIdentityBinding

```bash
kubectl apply -f podidentity-azureidentity.yaml
```

### Start Pod using proper label and try getting token

```bash
kubectl run --rm -it testpodidentity --image=alpine --labels="aadpodidbinding=avama2mi1"

curl -H Metadata:true -s --noproxy "*" "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https%3A%2F%2Fmanagement.azure.com%2F"
```

### Debug issues by looking at the logs of the binding

```bash
kubectl describe AzureIdentity avama2mi1

kubectl describe AzureIdentityBinding avama2mi1-binding
```

## Publisher can Read and Delete Cross-Tenant Role Assignments

```bash
# Publisher can read role assignments of the customer in a different tenant using REST API (but not yet the Azrue CLI) by passing tenantId parameter of the customer's tenant
az rest --method GET --uri "https://management.azure.com/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avama2-mrg/providers/Microsoft.Authorization/roleAssignments?api-version=2019-04-01-preview&tenantId=72f988bf-86f1-41af-91ab-2d7cd011db47&$filter=atScope%28%29" -o json

# Publisher can delete role assignments of the customer in a different tenant using REST API (but not yet the Azrue CLI) by passing tenantId parameter of the customer's tenant
az rest --method DELETE --uri "https://management.azure.com/subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourcegroups/avama2-mrg/providers/Microsoft.Authorization/roleAssignments/089ed05c-6e69-5d9a-b7c6-cdb12f2cc6e7?api-version=2019-04-01-preview&tenantId=72f988bf-86f1-41af-91ab-2d7cd011db47" -o json
```

## Refresh Azure Managed Application Permissions

As publisher, we can execute the following commands to read info about the managed app or to refresh its permissions.

```bash
# Publisher can read information about the managed app resource in the customer subscription
az rest --method GET --uri /subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avama2/providers/Microsoft.Solutions/applications/avama2managedapp?api-version=2019-07-01

# Refresh permissions for the managed app when "Customer allowed actions" or "Owner/Contributor" assigned is changed in the Partner Center by the publisher
az rest --method POST --uri /subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avama2/providers/Microsoft.Solutions/applications/avama2managedapp/refreshPermissions?api-version=2019-07-01
```

## Get Managed App Managed Identity

As publisher, we can execute the following command to get the managed app's system assigned managed identity access token for storage.

```bash
az rest --method POST --uri /subscriptions/c9c8ae57-acdb-48a9-99f8-d57704f18dee/resourceGroups/avama2/providers/Microsoft.Solutions/applications/avama2managedapp/listTokens?api-version=2018-09-01-preview --headers Content-Type=application/json --body "{authorizationAudience: 'https://storage.azure.com/'}" -o json
```
