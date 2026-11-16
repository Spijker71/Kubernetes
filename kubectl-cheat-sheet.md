# kubectl Command Cheat Sheet

Reference commands collected from the Kubernetes course notes. Replace every `<placeholder>` before running a command. Commands marked **Changes cluster state** create, update, scale, or roll back resources. Review the current context and namespace before running them.

## Context and namespace

```sh
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context>
kubectl config set-context --current --namespace=<namespace>
kubectl get namespaces
```

`kubectl config set-context` changes the default namespace in your local kubeconfig; it does not change the namespace of existing objects.

## Inspect resources

```sh
kubectl get pods -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl get deployments,replicasets,services -n <namespace>
kubectl get <resource> <name> -n <namespace> -o yaml
kubectl describe pod <pod> -n <namespace>
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

`kubectl get all` is a convenient overview, not a complete inventory. It omits resources such as Secrets, ConfigMaps, PVCs, and many custom resources.

## Rollouts and scaling

```sh
kubectl rollout status deployment/<deployment> -n <namespace>
kubectl rollout history deployment/<deployment> -n <namespace>
```

**Changes cluster state:** roll back the Deployment to its previous revision.

```sh
kubectl rollout undo deployment/<deployment> -n <namespace>
```

**Changes cluster state:** set a Deployment's replica count.

```sh
kubectl scale deployment/<deployment> --replicas=5 -n <namespace>
```

## Logs and container access

```sh
kubectl logs <pod> -n <namespace> --tail=100
kubectl logs <pod> -n <namespace> --follow
kubectl logs <pod> -c <container> -n <namespace> --tail=100
kubectl logs <pod> -n <namespace> --previous
kubectl exec -it <pod> -n <namespace> -- /bin/sh
kubectl cp <namespace>/<pod>:/path/in/container ./local-file
```

The container must include the requested shell. Avoid printing environment variables or Secret contents into shared terminals and logs.

## Configuration and manifests

```sh
kubectl get configmap <name> -n <namespace>
kubectl get secret <name> -n <namespace>
kubectl explain pod.spec.containers.envFrom.configMapRef
kubectl apply --dry-run=server -f <manifest.yaml>
kubectl diff -f <manifest.yaml>
kubectl kustomize <directory>
```

**Changes cluster state:** apply a manifest or a Kustomize directory.

```sh
kubectl apply -f <manifest.yaml>
kubectl apply -k <directory>
```

To generate a Secret manifest from a file without applying it:

```sh
kubectl create secret generic <secret-name> --from-file=<key>=<path> --dry-run=client -o yaml
```

The generated YAML contains secret data. Do not paste it into logs or commit it to source control.

## Services and networking

```sh
kubectl get services -n <namespace> -o wide
kubectl describe service <service> -n <namespace>
kubectl get endpointslices -n <namespace>
kubectl get ingress -n <namespace>
kubectl get gateways,httproutes -n <namespace>
```

Gateway API resources are available only when the corresponding CRDs and implementation are installed.

## KEDA

```sh
kubectl get crd scaledobjects.keda.sh
kubectl get scaledobjects,triggerauthentications -n <namespace>
kubectl describe scaledobject <scaledobject> -n <namespace>
kubectl get pods -n keda
kubectl logs deployment/keda-operator -n keda --tail=100
```

The KEDA CRDs must be installed before applying `ScaledObject` or `TriggerAuthentication` resources. Check the KEDA documentation for the version installed in the cluster.

## Storage

```sh
kubectl get persistentvolumes
kubectl get persistentvolumeclaims -A
kubectl describe pvc <claim> -n <namespace>
kubectl get pv <volume> -o yaml
```

Inspect `claimRef`, `phase`, `accessModes`, and `persistentVolumeReclaimPolicy` before changing or removing storage resources. A PV in `Released` is not necessarily ready to bind again. See [How to troubleshoot Kubernetes manifests and storage](how-to-troubleshoot-kubernetes-manifests-and-storage.md).

## AKS and access checks

```sh
az aks install-cli
az aks get-credentials --resource-group <resource-group> --name <cluster-name>
az aks show --resource-group <resource-group> --name <cluster-name> --query oidcIssuerProfile.issuerUrl -o tsv
kubectl auth can-i create pods/exec -n <namespace>
kubectl auth can-i get secrets -n <namespace>
```

`az aks install-cli` installs local Kubernetes CLI tools. `az aks get-credentials` updates the local kubeconfig. Confirm the selected context before using `kubectl` against a production cluster.

For workload identity and Key Vault CSI checks, see [How to troubleshoot AKS workload identity](how-to-troubleshoot-aks-workload-identity.md). For KEDA Service Bus scaling, see [How to configure KEDA Service Bus autoscaling](how-to-configure-keda-servicebus-autoscaling.md).
