# How to troubleshoot Kubernetes manifests and storage

Use this guide to narrow down Kustomize build failures, invalid Deployment templates, and PV/PVC binding issues. Inspect the rendered resources and storage policy before applying or deleting anything.

## Check a Kustomize build failure

Build the same base or overlay that Flux or your deployment workflow renders:

```sh
kubectl kustomize <kustomize-directory>
```

If the error says a resource ID is already registered, check the full resource identity (API group/version, kind, name, and namespace) across the Kustomize base and overlays. Remove duplicate resource references or use a patch/overlay for intentional customization; do not include the same resource once as a base and again as a standalone resource.

If the error says `file is not directory`, inspect the `resources` entries in each `kustomization.yaml`:

- A YAML manifest path should name a file and should not have a trailing slash.
- A base path should name a directory containing a `kustomization.yaml`.
- Resolve paths relative to the `kustomization.yaml` that declares them.

After correcting the paths or duplicate references, rebuild. Before deploying, you can ask the API server to validate the rendered resources:

```sh
kubectl apply --dry-run=server -k <kustomize-directory>
```

## Check a Deployment validation error

For errors such as `spec.template.spec.containers: Required value`, inspect the rendered Deployment and confirm the container list is nested at `spec.template.spec`:

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          image: <registry>/<image>:<tag>
          envFrom:
            - configMapRef:
                name: <config-map>
            - secretRef:
                name: <secret>
```

Every container entry needs a `name` and an `image`. Keep `envFrom`, `env`, ports, probes, and volume mounts under that container entry. If the API reports `containers[1].image` or another index, look for indentation that accidentally made a volume, environment source, or other field a second container.

Render first, then inspect the object that the API server will receive:

```sh
kubectl kustomize <kustomize-directory>
kubectl apply --dry-run=server -k <kustomize-directory>
```

## Investigate a PV/PVC binding conflict

PV names are cluster-scoped; PVC names are scoped to a namespace. Start with read-only checks:

```sh
kubectl get pvc -A
kubectl describe pvc <claim> -n <namespace>
kubectl get pv <volume> -o yaml
```

Inspect the PV's `claimRef`, `phase`, `accessModes`, `storageClassName`, and `persistentVolumeReclaimPolicy`. Confirm that the claim references the intended storage and that the PV is not bound to another claim. A PV is bound to one claim at a time; access modes describe how a volume can be mounted and do not generally allow multiple PVCs to bind the same PV.

A PV in `Released` is not the same as `Available`. With `Retain`, the claim has been released but the backing data is retained, and the PV is not automatically ready for a new claim. Decide whether the retained data must be preserved, backed up, or securely removed before any reuse.

Do not remove `kubernetes.io/pvc-protection`, force-delete the PVC, clear `claimRef`, or delete the PV as a first response. These actions can orphan workloads or expose/delete data. Follow the storage platform's recovery procedure and obtain the storage owner's approval before a destructive action.

For other common inspection commands, see the [kubectl command cheat sheet](kubectl-cheat-sheet.md).
