# How to troubleshoot AKS workload identity

Use this guide when a pod or the Azure Key Vault Secrets Store CSI Driver cannot authenticate with Microsoft Entra ID, when a CSI mount fails, or when an application reports an untrusted TLS certificate. Replace all placeholders with values from the target cluster. The original notes contained conflicting identity advice; this guide distinguishes Workload Identity from legacy node identity and Microsoft Entra pod-managed identity.

## 1. Confirm the identity flow

For Microsoft Entra Workload ID, a Kubernetes service-account token is federated with a user-assigned managed identity (UAMI). The service-account namespace and name are part of the federation subject. The workload and KEDA operator can use different service accounts and may require separate federated credentials.

For Key Vault CSI mounts, the Azure provider exchanges the projected token while mounting the volume. The application reads mounted files; it does not need to request an Entra token for that file-mount flow. If the application separately calls Azure APIs, it must use its own supported Azure Identity flow.

Do not mix Workload ID settings with legacy VM-managed identity or deprecated pod-managed identity settings. For current AKS setup and CSI parameter details, follow [AKS Workload Identity](https://learn.microsoft.com/azure/aks/workload-identity-overview) and [Key Vault CSI identity access](https://learn.microsoft.com/azure/aks/csi-secrets-store-identity-access#access-with-a-microsoft-entra-workload-id).

## 2. Check the cluster issuer and feature state

Confirm the cluster uses the expected subscription and that OIDC issuer and Workload Identity are enabled. The following query is read-only:

```powershell
az aks show --resource-group <aks-resource-group> --name <aks-cluster-name> --query "{oidc:oidcIssuerProfile.issuerUrl,workloadIdentity:securityProfile.workloadIdentity.enabled}" -o json
```

If either feature is not enabled, follow the AKS upgrade/change procedure and obtain cluster-owner approval. Enabling these features changes cluster configuration and is outside this troubleshooting guide's automatic actions.

## 3. Match the service account and federated credential

The Kubernetes service account should be in the same namespace as the workload and should use the intended UAMI client ID:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <service-account-name>
  namespace: <kubernetes-namespace>
  annotations:
    azure.workload.identity/client-id: "<uami-client-id>"
```

For a workload that uses the Workload Identity webhook, its Pod template must also have the required label and service account:

```yaml
spec:
  template:
    metadata:
      labels:
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: <service-account-name>
```

Check the actual deployed names and namespace:

```sh
kubectl get serviceaccount <service-account-name> -n <kubernetes-namespace> -o yaml
kubectl get pod <pod-name> -n <kubernetes-namespace> -o yaml
```

The federated credential values must match exactly:

- **Issuer:** the cluster's OIDC issuer URL.
- **Subject:** `system:serviceaccount:<kubernetes-namespace>:<service-account-name>`.
- **Audience:** `api://AzureADTokenExchange`.

For example, to create a credential for a UAMI using Azure CLI in PowerShell:

```powershell
az identity federated-credential create --name <credential-name> --identity-name <uami-name> --resource-group <uami-resource-group> --issuer <aks-oidc-issuer-url> --subject "system:serviceaccount:<kubernetes-namespace>:<service-account-name>" --audience api://AzureADTokenExchange
```

This command changes Azure identity configuration. Run it only after confirming the identity, namespace, subject, and issuer with the Azure owner. The credential resource name is an arbitrary unique identifier; it does not need to equal the service-account name.

## 4. Check the Key Vault `SecretProviderClass`

The CSI provider needs the expected Key Vault name, tenant ID, and identity client ID. For Workload Identity, current AKS examples use `usePodIdentity: "false"` with `clientID`; do not use `useVMManagedIdentity: "true"` to select Workload Identity.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: <secret-provider-class>
  namespace: <kubernetes-namespace>
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<uami-client-id>"
    keyvaultName: <key-vault-name>
    tenantId: <tenant-id>
    objects: |
      array:
        - |
          objectName: <key-vault-secret-name>
          objectType: secret
```

Check that the UAMI has the appropriate Key Vault data-plane permission for the object type. With Key Vault RBAC, the role differs for secrets versus keys/certificates. Use the least privilege required by the workload.

The `SecretProviderClass`, Pod, and any synchronized Kubernetes Secret must be in the intended namespace. The CSI mount is what causes the provider to retrieve the object. A Kubernetes Secret is not created merely by applying the `SecretProviderClass`.

If a Kubernetes Secret is required, configure `secretObjects` in the `SecretProviderClass` and ensure secret synchronization is enabled for the CSI driver. The synchronized Secret is materialized when a Pod mounts the CSI volume and is subject to normal Kubernetes Secret access controls. Prefer mounted files when the application supports them.

To inspect the installed `secretObjects` schema and a configured mapping, run:

```sh
kubectl explain secretproviderclass.spec.secretObjects --api-version=secrets-store.csi.x-k8s.io/v1
kubectl get secretproviderclass <secret-provider-class> -n <kubernetes-namespace> -o yaml
```

Each `secretObjects` entry maps a mounted provider object to a Kubernetes Secret. `secretName` names the Kubernetes Secret, `type` sets its Secret type, `data[].key` names a key in that Secret, and `data[].objectName` identifies the mounted object. The `objectName` must match an object in `spec.parameters.objects` (or its mounted alias, if one is configured).

To verify that synchronization created the Secret and check its key names without displaying values, run:

```sh
kubectl describe secret <secret-name> -n <kubernetes-namespace>
```

Do not share `kubectl get secret -o yaml` output: `.data` values are base64-encoded, not encrypted. Secret synchronization requires a Pod to mount the CSI volume; the Secret is removed after all Pods consuming it are deleted. If the Secret is missing, verify that a Pod mounts the intended `SecretProviderClass` and inspect its events.

## 5. Diagnose errors from the pod and CSI provider

Start with Pod status, events, and the CSI provider logs. Driver and provider pod names vary by AKS add-on and deployment:

```sh
kubectl describe pod <pod-name> -n <kubernetes-namespace>
kubectl get events -n <kubernetes-namespace> --sort-by=.metadata.creationTimestamp
kubectl get pods -A
kubectl logs <azure-provider-pod> -n <system-namespace> --tail=100
```

For `No matching federated identity record found`, compare the subject in the error to the federated credential. The error identifies which service account requested the token; it may be the KEDA operator rather than the application.

For `nodePublishSecretRef secret is not set`, verify that the CSI mount uses the intended identity method and that the provider configuration matches the installed driver version. Workload Identity does not use a Kubernetes Secret as an identity credential.

For `Secret not found`, check the Kubernetes namespace, whether CSI secret synchronization is configured, whether a Pod successfully mounted the CSI volume, and whether the expected Key Vault object exists.

## 6. Diagnose TLS trust errors in the container

For errors such as `x509: certificate signed by unknown authority`, check the certificate chain and the image trust store from inside the affected container:

```sh
kubectl exec -it <pod-name> -n <kubernetes-namespace> -- ls -l /etc/ssl/certs/ca-certificates.crt
kubectl exec -it <pod-name> -n <kubernetes-namespace> -- curl -sS https://<oidc-issuer-host>/.well-known/openid-configuration
```

If the container image lacks the public CA bundle, install `ca-certificates` at image build time. For Debian-based images, the package-install step is typically `apt-get update` followed by installing `ca-certificates`; remove package lists afterward and run the final image as a non-root user where practical. Installing a public CA bundle will not trust a private corporate CA; add only the approved CA chain through the image's documented trust-store process.

Do not download a root certificate from the endpoint that failed validation, disable TLS verification, or run the application as root just to work around certificate trust. If the certificate chain is incomplete or intercepted, involve the cluster/network owner.

References: [AKS Workload Identity overview](https://learn.microsoft.com/azure/aks/workload-identity-overview), [Key Vault CSI identity access](https://learn.microsoft.com/azure/aks/csi-secrets-store-identity-access#access-with-a-microsoft-entra-workload-id), [AKS Secrets Store CSI Driver](https://learn.microsoft.com/azure/aks/csi-secrets-store-driver), [CSI Driver secret synchronization](https://secrets-store-csi-driver.sigs.k8s.io/topics/sync-as-kubernetes-secret).
