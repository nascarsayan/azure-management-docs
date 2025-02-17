---
title: Use the Azure Key Vault Secret Store extension to sync secrets to the Kubernetes secret store for offline access in Azure Arc-enabled Kubernetes clusters
description: The Azure Key Vault Secret Store extension for Kubernetes ("Secret Store") automatically synchronizes secrets from an Azure Key Vault to a Kubernetes cluster for offline access.
ms.date: 09/26/2024
ms.topic: how-to
ms.custom: references_regions
---

# Use the Secret Store extension to fetch secrets for offline access in Azure Arc-enabled Kubernetes clusters

The Azure Key Vault Secret Store extension for Kubernetes ("Secret Store") automatically synchronizes secrets from an [Azure Key Vault](/azure/key-vault/general/overview) to an [Azure Arc-enabled Kubernetes cluster](overview.md) for offline access. This means you can use Azure Key Vault to store, maintain, and rotate your secrets, even when running your Kubernetes cluster in a semi-disconnected state. Synchronized secrets are stored in the cluster [secret store](https://Kubernetes.io/docs/concepts/configuration/secret/), making them available as Kubernetes secrets to be used in all the usual ways: mounted as data volumes, or exposed as environment variables to a container in a pod.

Synchronized secrets are critical business assets, so the Secret Store secures them through isolated namespaces and nodes, role-based access control (RBAC) policies, and limited permissions for the secrets synchronizer. For extra protection, [encrypt](https://Kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) the Kubernetes secret store on your cluster.

> [!TIP]
> The Secret Store extension is recommended for scenarios where offline access is necessary, or if you need secrets synced into the Kubernetes secret store. If you don't need these features, you can use the [Azure Key Vault Secrets Provider extension](tutorial-akv-secrets-provider.md) for secret management in your Arc-enabled Kubernetes clusters. It is not recommended to run both the online Azure Key Vault Secrets Provider extension and the offline Secret Store extension side-by-side in a cluster.

This article shows you how to install and configure the Secret Store as an [Azure Arc-enabled Kubernetes extension](conceptual-extensions.md).

> [!IMPORTANT]
> Secret Store is currently in PREVIEW.
> See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for legal terms that apply to Azure features that are in beta, preview, or otherwise not yet released into general availability.

## Prerequisites

- A cluster [connected to Azure Arc](quickstart-connect-cluster.md), running Kubernetes version 1.27 or higher, and in one of the supported regions (East US, East US2, West US, West US2, West US3, West Europe, North Europe). The region is defined by the resource group region used for creating the Arc cluster.
- The examples throughout this guide use a [K3s](https://k3s.io/) cluster.
- Ensure you meet the [general prerequisites for cluster extensions](extensions.md#prerequisites), including the latest version of the `k8s-extension` Azure CLI extension.
- cert-manager is required to support TLS for intracluster log communication. The examples later in this guide direct you though installation. For more information about cert-manager, see [cert-manager.io](https://cert-manager.io/)

Before you begin, set environment variables to be used for configuring Azure and cluster resources. If you already have a managed identity, Azure Key Vault, or other resource listed here, update the names in the environment variables to reflect those resources.

```azurecli
az login
export RESOURCE_GROUP="oidc-issuer"
export LOCATION="westus2"
export AZURE_STORAGE_ACCOUNT="oidcissuer$(openssl rand -hex 4)"
export AZURE_STORAGE_CONTAINER="oidc-test"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
export AZURE_TENANT_ID="$(az account show -s $SUBSCRIPTION --query tenantId --output tsv)"
export CURRENT_USER="$(az ad signed-in-user show --query userPrincipalName --output tsv)"
export KEYVAULT_NAME="azwi-kv-$(openssl rand -hex 4)"
export KEYVAULT_SECRET_NAME="my-secret"
export USER_ASSIGNED_IDENTITY_NAME="myIdentity"
az account set --subscription "${SUBSCRIPTION}"
export SERVICE_ACCOUNT_ISSUER="https://$AZURE_STORAGE_ACCOUNT.blob.core.windows.net/${AZURE_STORAGE_CONTAINER}"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="myFedIdentity"
export KUBERNETES_NAMESPACE="default"
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
```

## Configure an identity to access secrets

To access and synchronize a given Azure Key Vault secret, the Secret Store requires access to an Azure managed identity with appropriate Azure permissions to access that secret. The managed identity must be linked to a Kubernetes service account through [federation](/graph/api/resources/federatedidentitycredentials-overview). The Kubernetes service account is what you use in a Kubernetes pod or other workload to access secrets from the Kubernetes secret store. The Secret Store extension uses the associated federated Azure managed identity to pull secrets from Azure Key Vault to your Kubernetes secret store. The following sections describe how to set this up.

### Host OIDC public information about your cluster's Service Account issuer

Use of federated identity currently requires you to set up cloud storage to host OIDC format information about the public keys of your cluster's Service Account issuer. In this section, you set up a secured, public Open ID Connect (OIDC) issuer URL using Azure blob storage, then upload a minimal discovery document to the storage account. For background, see [OIDC configuration for self-managed clusters](https://azure.github.io/azure-workload-identity/docs/installation/self-managed-clusters.html).

1. Create an Azure storage account.

   ``` console
   az group create --name "${RESOURCE_GROUP}" --location "${LOCATION}"
   az storage account create --resource-group "${RESOURCE_GROUP}" --name "${AZURE_STORAGE_ACCOUNT}" --allow-blob-public-access true
   az storage container create --name "${AZURE_STORAGE_CONTAINER}" --public-access blob
    ```

   > [!NOTE]
   > 'az storage account create' may fail if your Azure instance hasn't enabled the "Microsoft.Storage" service.
   > If you hit a failure [register the Microsoft.Storage resource provider in your subscription](/azure/azure-resource-manager/management/resource-providers-and-types#register-resource-provider).

1. Generate a discovery document. Upload it the storage account, and then verify that it's publicly accessible.

   ```bash
   cat <<EOF > openid-configuration.json
   {
     "issuer": "https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/${AZURE_STORAGE_CONTAINER}/",
     "jwks_uri": "https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/${AZURE_STORAGE_CONTAINER}/openid/v1/jwks",
     "response_types_supported": [
       "id_token"
     ],
     "subject_types_supported": [
       "public"
     ],
     "id_token_signing_alg_values_supported": [
       "RS256"
     ]
   }
   EOF
   ```

   ``` console
   az storage blob upload --container-name "${AZURE_STORAGE_CONTAINER}" --file openid-configuration.json --name .well-known/openid-configuration
   curl -s "https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/${AZURE_STORAGE_CONTAINER}/.well-known/openid-configuration"
   ```

1. Obtain the public key for your cluster's service account issuer from its private key. You'll likely need to run this command as a superuser. The following example is for k3s. Your cluster may store the service account issuer private key at a different location.

   ``` console
   sudo openssl rsa -in /var/lib/rancher/k3s/server/tls/service.key -pubout -out sa.pub
   ```

1. Download the latest [azwi command line tool](https://github.com/Azure/azure-workload-identity/releases), which you can use to create a JWKS document from the public key, and untar it:

   ``` console
   tar -xzf <path to downloaded azwi tar.gz>
   ```
  
1. Generate the JWKS document. Upload it to the storage account, and then verify that it's publicly accessible.

   ``` console
   ./azwi jwks --public-keys sa.pub --output-file jwks.json
   az storage blob upload --container-name "${AZURE_STORAGE_CONTAINER}" --file jwks.json --name openid/v1/jwks
   curl -s "https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/${AZURE_STORAGE_CONTAINER}/openid/v1/jwks"
   ```

### Configure your cluster's Service Account token issuer with the hosted URL

Your cluster must also be configured to issue Service Account tokens with an issuer URL (`service-account-issuer`) field that points to the storage account you created in the previous section. For more background on the federation configuration, see [cluster configuration for an OIDC issuer](https://azure.github.io/azure-workload-identity/docs/installation/self-managed-clusters/configurations.html).

Optionally, you can configure limits on the Secret Store's own permissions as a privileged resource running in the control plane by configuring [`OwnerReferencesPermissionEnforcement`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#ownerreferencespermissionenforcement) [admission controller](https://Kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#how-do-i-turn-on-an-admission-controller). This admission controller constrains how much the Secret Store can change other objects in the cluster.

Your Kubernetes cluster must be running Kubernetes version 1.27 or higher.

1. Configure your [kube-apiserver](https://Kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) with the issuer URL field and permissions enforcement. The following example is for a k3s cluster. Your cluster may have different means for changing API server arguments: `--kube-apiserver-arg="--service-account-issuer=${SERVICE_ACCOUNT_ISSUER}" and --kube-apiserver-arg="--enable-admission-plugins=OwnerReferencesPermissionEnforcement"`.

   - Get the service account issuer url.

     ```console
      echo $SERVICE_ACCOUNT_ISSUER
      ```

   - Open the K3s server configuration file.

     ```console
      sudo nano /etc/systemd/system/k3s.service
      ```

   - Edit the server configuration to look like the following example, replacing <SERVICE_ACCOUNT_ISSUER> with the output from `echo $SERVICE_ACCOUNT_ISSUER`.

      ```console
      ExecStart=/usr/local/bin/k3s \
       server --write-kubeconfig-mode=644 \
          --kube-apiserver-arg="--service-account-issuer=<SERVICE_ACCOUNT_ISSUER>" \
          --kube-apiserver-arg="--enable-admission-plugins=OwnerReferencesPermissionEnforcement"
      ```

1. Restart your kube-apiserver.

    ```console
   sudo systemctl daemon-reload
   sudo systemctl restart k3s
   ```

### Create an Azure Key Vault

[Create an Azure Key Vault](/azure/key-vault/secrets/quick-create-cli) and add a secret. If you already have an Azure Key Vault and secret, you can skip this section.

1. Create an Azure Key Vault:

   ```azurecli
   az keyvault create --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --name "${KEYVAULT_NAME}" --enable-rbac-authorization
   ```

1. Give yourself 'Secrets Officer' permissions on the vault, so you can create a secret:

   ```azurecli
   az role assignment create --role "Key Vault Secrets Officer" --assignee ${CURRENT_USER} --scope /subscriptions/${SUBSCRIPTION}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${KEYVAULT_NAME}
   ```

1. Create a secret and update it so you have two versions:

   ```azurecli
   az keyvault secret set --vault-name "${KEYVAULT_NAME}" --name "${KEYVAULT_SECRET_NAME}" --value 'Hello!'
   az keyvault secret set --vault-name "${KEYVAULT_NAME}" --name "${KEYVAULT_SECRET_NAME}" --value 'Hello2'
   ```

### Create a user-assigned managed identity

Next, create a user-assigned managed identity and give it permissions to access the Azure Key Vault. If you already have a managed identity with Key Vault Reader and Key Vault Secrets User permissions to the Azure Key Vault, you can skip this section. For more information, see [Create a user-assigned managed identities](/entra/identity/managed-identities-azure-resources/how-manage-user-assigned-managed-identities?pivots=identity-mi-methods-azp#create-a-user-assigned-managed-identity) and [Using Azure RBAC secret, key, and certificate permissions with Key Vault](/azure/key-vault/general/rbac-guide?tabs=azure-cli#using-azure-rbac-secret-key-and-certificate-permissions-with-key-vault).

1. Create the user-assigned managed identity:

   ```azurecli
   az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --subscription "${SUBSCRIPTION}"
   ```

1. Give the identity Key Vault Reader and Key Vault Secrets User permissions. You may need to wait a moment for replication of the identity creation before these commands succeed:

   ```azurecli
   export USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' -otsv)"
   az role assignment create --role "Key Vault Reader" --assignee "${USER_ASSIGNED_CLIENT_ID}" --scope /subscriptions/${SUBSCRIPTION}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${KEYVAULT_NAME}
   az role assignment create --role "Key Vault Secrets User" --assignee "${USER_ASSIGNED_CLIENT_ID}" --scope /subscriptions/${SUBSCRIPTION}/resourcegroups/${RESOURCE_GROUP}/providers/Microsoft.KeyVault/vaults/${KEYVAULT_NAME}
   ```

### Create a federated identity credential

Create a Kubernetes service account for the workload that needs access to secrets. Then, create a [federated identity credential](https://azure.github.io/azure-workload-identity/docs/topics/federated-identity-credential.html) to link between the managed identity, the OIDC service account issuer, and the Kubernetes Service Account.

1. Create a Kubernetes Service Account that will be federated to the managed identity. Annotate it with details of the associated user-assigned managed identity.

   ``` console
   kubectl create ns ${KUBERNETES_NAMESPACE}
   ```

   ``` console
   cat <<EOF | kubectl apply -f -
     apiVersion: v1
     kind: ServiceAccount
     metadata:
       name: ${SERVICE_ACCOUNT_NAME}
       namespace: ${KUBERNETES_NAMESPACE}
   EOF
   ```

1. Create a federated identity credential:

   ```azurecli
   az identity federated-credential create --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name ${USER_ASSIGNED_IDENTITY_NAME} --resource-group ${RESOURCE_GROUP} --issuer ${SERVICE_ACCOUNT_ISSUER} --subject system:serviceaccount:${KUBERNETES_NAMESPACE}:${SERVICE_ACCOUNT_NAME}
   ```

## Install and use the Secret Store

The Secret Store is available as an Azure Arc extension. An [Azure Arc-enabled Kubernetes cluster](overview.md) can be extended with [Azure Arc-enabled Kubernetes extensions](conceptual-extensions.md). Extensions enable Azure capabilities on your connected cluster and provide an Azure Resource Manager-driven experience for the extension installation and lifecycle management.

The Secret Store is installed as an [Azure Arc extension](extensions.md)

### Install cert-manager and trust-manager

[cert-manager](https://cert-manager.io/) and [trust-manager](https://cert-manager.io/docs/trust/trust-manager/) are required for secure communication of logs between cluster services and must be installed before the Arc extension.

1. Install cert-manager.

   ```azurecli
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
   ```

1. Install trust-manager.

   ```azurecli
   helm repo add jetstack https://charts.jetstack.io
   helm repo update
   helm upgrade trust-manager jetstack/trust-manager --install --namespace cert-manager --wait
   ```

### Install the Secret Store Azure Arc extension

Be sure that your Kubernetes cluster is [connected to Azure Arc](quickstart-connect-cluster.md) before installing the extension.

1. Set two environment variables for the resource group and name of your connected cluster. If you followed the quickstart linked earlier, these are 'AzureArcTest' and 'AzureArcTest1' respectively.

   ```azurecli
   export ARC_RESOURCE_GROUP="AzureArcTest"
   export ARC_CLUSTER_NAME="AzureArcTest1"
   ```

1. Install the Secret Store extension to your Arc-enabled cluster using the following command:

   ``` console
   az k8s-extension create \
     --cluster-name ${ARC_CLUSTER_NAME} \
     --cluster-type connectedClusters \
     --extension-type microsoft.azure.secretstore \
     --resource-group ${ARC_RESOURCE_GROUP} \
     --release-train preview \
     --name ssarcextension \
     --scope cluster 
   ```

   If desired, you can optionally modify the default rotation poll interval by adding `--configuration-settings rotationPollIntervalInSeconds=<time_in_seconds>`:

   | Parameter name                    | Description                         | Default value                         |
   |---------------------------------|-------------------------------------------------------------------------------|----------------------------------------------|
   | `rotationPollIntervalInSeconds`          | Specifies how quickly the Secret Store  checks or updates the secret it's managing.       | `3600` (1 hour)                                             |

## Configure the Secret Store

Configure the installed extension with information about your Azure Key Vault and which secrets to synchronize to your cluster by defining instances of Kubernetes [custom resources](https://Kubernetes.io/docs/concepts/extend-Kubernetes/api-extension/custom-resources/). You create two types of custom resources:

- A `SecretProviderClass` object to define the connection to the Key Vault.
- A `SecretSync` object for each secret to be synchronized.

### Create a `SecretProviderClass` resource

The `SecretProviderClass` resource is used to define the connection to the Azure Key Vault, the identity to use to access the vault, which secrets to synchronize, and the number of versions of each secret to retain locally.

You need a separate `SecretProviderClass` for each Azure Key Vault you intend to synchronize, for each identity used for access to an Azure Key Vault, and for each target Kubernetes namespace.

Create one or more `SecretProviderClass` YAML files with the appropriate values for your Key Vault and secrets by following this example.

``` yaml
cat <<EOF > spc.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: secret-provider-class-name                      # Name of the class; must be unique per Kubernetes namespace
  namespace: ${KUBERNETES_NAMESPACE}                    # Kubernetes namespace to make the secrets accessible in
spec:
  provider: azure
  parameters:
    clientID: "${USER_ASSIGNED_CLIENT_ID}"               # Managed Identity Client ID for accessing the Azure Key Vault with.
    keyvaultName: ${KEYVAULT_NAME}                       # The name of the Azure Key Vault to synchronize secrets from.
    objects: |
      array:
        - |
          objectName: ${KEYVAULT_SECRET_NAME}            # The name of the secret to sychronize.
          objectType: secret
          objectVersionHistory: 2                       # [optional] The number of versions to synchronize, starting from latest.
    tenantID: "${AZURE_TENANT_ID}"                       # The tenant ID of the Key Vault 
EOF
```

### Create a `SecretSync` object

Each synchronized secret also requires a `SecretSync` object, to define cluster-specific information. Here you specify information such as the name of the secret in your cluster and names for each version of the secret stored in your cluster.

Create one `SecretSync` object YAML file for each secret, following this template. The Kubernetes namespace should match the namespace of the matching `SecretProviderClass`.

```yaml
cat <<EOF > ss.yaml
apiVersion: secret-sync.x-k8s.io/v1alpha1
kind: SecretSync
metadata:
  name: secret-sync-name                                  # Name of the object; must be unique per Kubernetes namespace
  namespace: ${KUBERNETES_NAMESPACE}                      # Kubernetes namespace
spec:
  serviceAccountName: ${SERVICE_ACCOUNT_NAME}             # The Kubernetes service account to be given permissions to access the secret.
  secretProviderClassName: secret-provider-class-name     # The name of the matching SecretProviderClass with the configuration to access the AKV storing this secret
  secretObject:
    type: Opaque
    data:
    - sourcePath: ${KEYVAULT_SECRET_NAME}/0                # Name of the secret in Azure Key Vault with an optional version number (defaults to latest)
      targetKey: ${KEYVAULT_SECRET_NAME}-data-key0         # Target name of the secret in the Kubernetes Secret Store (must be unique)
    - sourcePath: ${KEYVAULT_SECRET_NAME}/1                # [optional] Next version of the AKV secret. Note that versions of the secret must match the configured objectVersionHistory in the secrets provider class 
      targetKey: ${KEYVAULT_SECRET_NAME}-data-key1         # [optional] Next target name of the secret in the K8s Secret Store
EOF
```

### Apply the configuration CRs

Apply the configuration custom resources (CRs) using the `kubectl apply` command:

``` bash
kubectl apply -f ./spc.yaml
kubectl apply -f ./ss.yaml
```

The Secret Store automatically looks for the secrets and begins syncing them to the cluster.

### View configuration options

To view additional configuration options for these two custom resource types, use the `kubectl describe` command to inspect the CRDs in the cluster:

```bash
# Get the name of any applied CRD(s)
kubectl get crds -o custom-columns=NAME:.metadata.name

# View the full configuration options and field parameters for a given CRD
kubectl describe crd secretproviderclass
kubectl describe crd secretsync
```

## Observe secrets synchronizing to the cluster

Once the configuration is applied, secrets begin syncing to the cluster automatically at the cadence specified when installing the Secret Store.

### View synchronized secrets

View the secrets synchronized to the cluster by running the following command:

```bash
# View a list of all secrets in the namespace
kubectl get secrets -n ${KUBERNETES_NAMESPACE}

# View details of all secrets in the namespace
kubectl get secrets -n ${KUBERNETES_NAMESPACE} -o yaml
```

### View last sync status

To view the status of the most recent synchronization for a given secret, use the `kubectl describe` command for the `SecretSync` object. The output includes the secret creation timestamp, the versions of the secret, and detailed status messages for each synchronization event. This output can be used to diagnose connection or configuration errors, and to observe when the secret value changes.

```bash
kubectl describe secretsync secret-sync-name -n ${KUBERNETES_NAMESPACE}
```

### View secrets values

To view the synchronized secret values, now stored in the Kubernetes secret store, use the following command:

```bash
kubectl get secret secret-sync-name -n ${KUBERNETES_NAMESPACE} -o jsonpath="{.data.${KEYVAULT_SECRET_NAME}-data-key0}" | base64 -d
kubectl get secret secret-sync-name -n ${KUBERNETES_NAMESPACE} -o jsonpath="{.data.${KEYVAULT_SECRET_NAME}-data-key1}" | base64 -d
```

## Troubleshooting

The Secret Store is a Kubernetes deployment that contains a pod with two containers: the controller, which manages storing secrets in the cluster, and the provider, which manages access to, and pulling secrets from, the Azure Key Vault. Each synchronized secret has a `SecretSync` object that contains the status of the synchronization of that secret from Azure Key Vault to the cluster secret store.

To troubleshoot an issue, start by looking at the state of the `SecretSync` object, as described in [View last sync status](#view-last-sync-status). The following table lists common status types, their meanings, and potential troubleshooting steps to resolve errors.

| SecretSync Status Type     | Details      | Steps to fix/investigate further    |
|------------|--------------|-------------------------------------|
| `CreateSucceeded` | The secret was created successfully. | n/a |
| `CreateFailedProviderError` | Secret creation failed due to some issue with the provider (connection to Azure Key Vault). This failure could be due to internet connectivity, insufficient permissions for the identity syncing secrets, misconfiguration of the `SecretProviderClass`, or other issues. | Investigate further by looking at the logs of the provider using the following commands: <br>```kubectl get pods -n azure-secret-store``` <br>```kubectl logs <secret-sync-controller-pod-name> -n azure-secret-store --container='provider-azure-installer'``` |
| `CreateFailedInvalidLabel` | The secret creation failed because the secret already exists without the correct Kubernetes label that the Secret Store uses to manage its secrets.| Remove the existing label and secret and allow the Secret Store to recreate the secret: ```kubectl delete secret <secret-name>``` <br>To force the Secret Store to recreate the secret faster than the configured rotation poll interval, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```) and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `CreateFailedInvalidAnnotation` | Secret creation failed because the secret already exists without the correct Kubernetes annotation that the Secret Store uses to manage its secrets. | Remove the existing annotation and secret and allow the Secret Store to recreate the secret: ```kubectl delete secret <secret-name>``` <br>To force the Secret Store to recreate the secret faster than the configured rotation poll interval, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```) and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `UpdateNoValueChangeSucceeded` | The Secret Store checked Azure Key Vault for updates at the end of the configured poll interval, but there were no changes to sync. | n/a |
| `UpdateValueChangeOrForceUpdateSucceeded` | The Secret Store checked Azure Key Vault for updates and successfully updated the value. | n/a |
| `UpdateFailedInvalidLabel` | Secret update failed because the label on the secret that the Secret Store uses to manage its secrets was modified. | Remove the existing label and secret, and allow the Secret Store to recreate the secret: ```kubectl delete secret <secret-name>``` <br>To force the Secret Store to recreate the secret faster than the configured rotation poll interval, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```) and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `UpdateFailedInvalidAnnotation` | Secret update failed because the annotation on the secret that the Secret Store uses to manage its secrets was modified. | Remove the existing annotation and secret and allow the Secret Store to recreate the secret: ```kubectl delete secret <secret-name>``` <br>To force the Secret Store to recreate the secret faster than the configured rotation poll interval, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```) and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `UpdateFailedProviderError` | Secret update failed due to some issue with the provider (connection to Azure Key Vault). This failure could be due to internet connectivity, insufficient permissions for the identity syncing secrets, configuration of the `SecretProviderClass`, or other issues. | Investigate further by looking at the logs of the provider using the following commands: <br>```kubectl get pods -n azure-secret-store``` <br>```kubectl logs <secret-sync-controller-pod-name> -n azure-secret-store --container='provider-azure-installer'``` |
| `UserInputValidationFailed` | Secret update failed because the secret sync class was configured incorrectly (such as an invalid secret type). | Review the secret sync class definition and correct any errors. Then, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```), delete the secret sync class (```kubectl delete -f <path_to_secret_sync>```), and reapply the secret sync class (```kubectl apply -f <path_to_secret_sync>```). |
| `ControllerSpcError` | Secret update failed because the Secret Store failed to get the provider class or the provider class is misconfigured. | Review the provider class and correct any errors. Then, delete the `SecretSync` object (```kubectl delete secretsync <secret-name>```), delete the provider class (```kubectl delete -f <path_to_provider>```), and reapply the provider class (```kubectl apply -f <path_to_provider>```). |
| `ControllerInternalError` | Secret update failed due to an internal error in the Secret Store. | Check the Secret Store logs or the events for more information: <br>```kubectl get pods -n azure-secret-store``` <br>```kubectl logs <secret-sync-controller-pod-name> -n azure-secret-store --container='manager'``` |
| `SecretPatchFailedUnknownError` | Secret update failed during patching the Kubernetes secret value. This failure might occur if the secret was modified by someone other than the Secret Store or if there were issues during an update of the Secret Store. | Try deleting the secret and `SecretSync` object, then let the Secret Store recreate the secret by reapplying the secret sync CR: <br>```kubectl delete secret <secret-name>``` <br>```kubectl delete secretsync <secret-name>```  <br>```kubectl apply -f <path_to_secret_sync>``` |

## Remove the Secret Store

To remove the Secret Store and stop synchronizing secrets, uninstall it with the `az k8s-extension delete` command:

```console
az k8s-extension delete --name ssarcextension --cluster-name $ARC_CLUSTER_NAME  --resource-group $RESOURCE_GROUP  --cluster-type connectedClusters    
```

Uninstalling the extension doesn't remove secrets, `SecretSync` objects, or CRDs from the cluster. These objects must be removed directly with `kubectl`.

Deleting the SecretSync CRD removes all `SecretSync` objects, and by default removes all owned secrets, but secrets may persist if:

- You modified ownership of any of the secrets.
- You changed the [garbage collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) settings in your cluster, including setting different [finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/).

In the above cases, secrets must be deleted directly using `kubectl`.

## Next steps

- Learn more about [Azure Arc extensions](extensions.md).
- Learn more about [Azure Key Vault](/azure/key-vault/general/overview).
