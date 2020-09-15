# Sample Azure Managed Application with AKS

## Testing prior to publishing in marketplace

While developing and iterating, it is convenient to test the app without having to publish it in marketplace every time since push to preview may sometime take a few hours.

> When testing locally, we need to comment out the "delegatedManagedIdentityResourceId" lines in the role assignments since this property can be used only in cross-tenant scenarios such as deployment from Azure Marketplace.

Test the application using <http://github.com/arm-ttk> tool

```cmd
C:\Code\GitHub\Azure\arm-ttk\arm-ttk\Test-AzTemplate.cmd
```

Zip relevant files and upload to a storage container

```bash
tar -a -c -f ama-aks.zip createUiDefinition.json mainTemplate.json viewDefinition.json

azcopy copy ./ama-aks.zip "https://YOUR_STORAGE_ACCOUNT.blob.core.windows.net/YOUR_STORAGE_CONTAINER/ama-aks.zip?SHARED_ACCESS_SIGNATURE_WITH_WRITE_PERMISSION"
```

Create managed app definition for testing the deployment

```bash
az group create --name avamaaks --location eastus

az managedapp definition create --name "azure-managed-app-aks" --location eastus --resource-group avamaaks --lock-level ReadOnly --display-name "Azure Managed App AKS" --description "Azure Managed App AKS Example" --authorizations "YOUR_AAD_GROUP_PRINCIPAL_ID:b24988ac-6180-42a0-ab88-20f7382dd24c" --package-file-uri "https://YOUR_STORAGE_ACCOUNT.blob.core.windows.net/ama-aks/ama-aks.zip"
```

Deploy managed application from the definition created above using <https://portal.azure.com>

## Testing after publishing in marketplace

After the app seems to work, publish in your [Partner Center account](https://docs.microsoft.com/en-us/azure/azure-resource-manager/managed-applications/publish-marketplace-app) by uploading the ama-aks.zip file to the relevant plan's technical configuration.

After the app is available in preview, search for the app in <https://poral.azure.com> "Create a resource", and deploy using the UI.
