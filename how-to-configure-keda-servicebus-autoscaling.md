# How to configure KEDA Service Bus autoscaling

Use this guide to connect a KEDA scaler to an Azure Service Bus queue and scale a long-running Kubernetes workload. Check the installed KEDA version before applying examples; the authentication API and supported fields can vary by version.

## Choose the scaling resource

| KEDA resource | Target | Use it for |
| --- | --- | --- |
| `ScaledObject` | A Deployment or StatefulSet | Scaling a continuously running service or worker up and down |
| `ScaledJob` | A Job template | Creating event-driven batch work that runs to completion |

`scaleTargetRef.name` is the metadata name of the Deployment or StatefulSet, not a Pod name. A ScaledJob instead contains a Job template.

## Check KEDA is installed

```sh
kubectl get pods -n <keda-namespace>
kubectl get crd scaledobjects.keda.sh
kubectl get crd triggerauthentications.keda.sh
```

If the CRD is missing, install or reconcile a KEDA release compatible with the cluster before applying KEDA custom resources. Use the current [KEDA installation guide](https://keda.sh/docs/latest/deploy/) and pin the release used by your environment; do not apply an old release manifest from these notes blindly.

## Configure a Service Bus queue trigger

For Azure Workload Identity, KEDA's Azure Service Bus scaler uses a `TriggerAuthentication` with `podIdentity.provider: azure-workload`. The Service Bus scaler runs in the KEDA operator, so federation must match the identity and Kubernetes service account actually used by that operator. Do not assume the application's service account is also used by the scaler. KEDA installation and identity wiring are version- and cluster-specific.

The following example is a template. Replace the queue, Service Bus namespace, Kubernetes namespace, scale target, and authentication names. The `namespace` under trigger metadata is the **Azure Service Bus namespace name**, not the Kubernetes namespace.

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: servicebus-workload-auth
  namespace: <kubernetes-namespace>
spec:
  podIdentity:
    provider: azure-workload
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: servicebus-queue-scaler
  namespace: <kubernetes-namespace>
spec:
  scaleTargetRef:
    kind: Deployment
    name: <deployment-name>
  minReplicaCount: 0
  maxReplicaCount: 5
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: <queue-name>
        namespace: <servicebus-namespace-name>
        messageCount: "5"
        cloud: AzurePublicCloud
      authenticationRef:
        name: servicebus-workload-auth
```

`messageCount` is the approximate queue message threshold for scaling. Set `minReplicaCount`, `maxReplicaCount`, polling behavior, and threshold according to the workload's latency and capacity requirements.

Apply the manifests only after confirming the KEDA CRDs and identity configuration:

```sh
kubectl apply --dry-run=server -f <keda-manifest.yaml>
kubectl apply -f <keda-manifest.yaml>
```

The second command changes cluster state.

## Troubleshoot common failures

### The API does not recognize `ScaledObject`

Confirm the CRD exists and that the controller is ready. Ensure KEDA is installed before Flux or another GitOps reconciler applies KEDA resources. Check the deployed release's API versions rather than relying on an old example.

### The scaler cannot authenticate

Inspect the identity named in the error. The federated credential must match all of these values:

- The OIDC issuer URL from the AKS cluster.
- The exact subject, `system:serviceaccount:<keda-namespace>:<keda-operator-service-account>` (use the actual KEDA operator namespace and service-account name).
- The audience `api://AzureADTokenExchange`.

The managed identity also needs the required Azure permissions to read Service Bus metrics. Review the [KEDA Azure Service Bus scaler documentation](https://keda.sh/docs/latest/scalers/azure-service-bus/) and your identity configuration before changing either service account.

### The target or trigger is not being read

Check the object name and namespace, then inspect the KEDA operator logs:

```sh
kubectl describe scaledobject <scaledobject-name> -n <kubernetes-namespace>
kubectl logs deployment/<keda-operator-deployment> -n <keda-namespace> --tail=100
```

ScaledObjects do not produce container logs themselves. The operator logs and the ScaledObject events/status contain scaler diagnostics.

References: [KEDA Azure Service Bus scaler](https://keda.sh/docs/latest/scalers/azure-service-bus/), [KEDA authentication](https://keda.sh/docs/latest/authentication/).
