# Plan: Manual Excel-conversion CPU and Memory Overrides

**Status: Proposed. This plan has not been applied to the manifests or clusters.**

## Goal and scope

Allow operators to temporarily set CPU and memory for the Excel-conversion KEDA ScaledJob without Flux reverting those manual changes. Restore its environment defaults every day at midnight in the `Europe/Amsterdam` time zone.

The planned scope is `excelconversion-scaler` in ACC, DEV, TST, and PRD, in both the root `Flux/` tree and `DNB.Datalogistiek.Kubernetes/Flux/`. Cryptography and other ScaledJobs are out of scope.

## Proposed design

1. Configure each owning Flux `Kustomization` to ignore only these four fields on the matching `keda.sh/v1alpha1` `ScaledJob` named `excelconversion-scaler`:
   - `/spec/jobTargetRef/template/spec/containers/0/resources/requests/cpu`
   - `/spec/jobTargetRef/template/spec/containers/0/resources/requests/memory`
   - `/spec/jobTargetRef/template/spec/containers/0/resources/limits/cpu`
   - `/spec/jobTargetRef/template/spec/containers/0/resources/limits/memory`
2. Keep `ephemeral-storage` and all other fields under Flux reconciliation.
3. Add a dedicated ServiceAccount, Role, RoleBinding, and CronJob per target namespace. Restrict the Role to `get` and `patch` on the named `scaledjobs.keda.sh` resource `excelconversion-scaler` in that namespace.
4. Schedule the CronJob as `0 0 * * *` with `timeZone: Europe/Amsterdam` and `concurrencyPolicy: Forbid`. Patch only the four CPU/memory fields back to that environment's defaults.
5. Use an organization-approved kubectl image pinned to a version or digest. Do not use the placeholder image in the current example.

## Reset defaults

The values below are from the existing Excel-conversion environment patches. CPU and memory are listed as request followed by limit.

| Environment | CPU request / limit | Memory request / limit |
| --- | --- | --- |
| ACC | `2000m` / `2000m` | `60Gi` / `60Gi` |
| DEV | `2048m` / `2048m` | `8Gi` / `16Gi` |
| TST | `2048m` / `2048m` | `8Gi` / `16Gi` |
| PRD | `4000m` / `4000m` | `105Gi` / `115Gi` |

If an environment's Git defaults change, update its CronJob reset values in the same change.

## Implementation sequence

1. Locate the Git-managed Flux `Kustomization` or Kustomizations reconciling the two manifest trees. Verify the installed Flux CRD and controller support the `spec.ignore` rules before changing them.
2. Add the four narrowly scoped ignore paths to the owning Flux definition(s), matching the actual resource name and target environment namespace(s).
3. Add the reset CronJob and dedicated namespace-scoped RBAC in both manifest trees. Register shared resources and environment-specific reset values through the existing base and overlay Kustomize structure.
4. Build all four overlays in both trees and review the rendered resource names, namespaces, reset values, and RBAC.
5. Test in DEV: manually change CPU/memory, confirm a Flux reconciliation preserves the override, run the reset Job, and confirm the defaults return while ephemeral-storage is unchanged. Roll out to other environments only after this check passes.

## Prerequisites and blockers

- The Git-managed Flux owner definition was not found in the workspace, and cluster permissions previously prevented discovery of the owner. A sample Flux Kustomization is not a substitute for the live Git-managed definition.
- The installed Flux controller/CRD version must be checked for support of `spec.ignore`.
- The DEV AKS cluster was confirmed as Kubernetes `1.36.5` with KEDA enabled. Kubernetes versions for ACC, TST, and PRD still need checking for CronJob `timeZone` support.
- An organization-approved, pinned kubectl image has not yet been identified.

Do not proceed with cluster changes until the Flux owner and controller support are confirmed, and the approved image is selected.

## Example file status

- `Flux/flux-kustomization-ignore.example.yml` now targets `excelconversion-scaler` in the DEV namespace.
- `Flux/excelconversion-resource-reset-cronjob.example.yml` targets the Excel-conversion ScaledJob and uses the DEV Excel-conversion defaults with corrected resource names.
- The CronJob image remains a placeholder until an organization-approved, compatible image is selected.

The examples are illustrative only; the owning Flux definition and deployment prerequisites above still need to be verified before applying this plan.

## References

- [Flux Kustomization ignore rules](https://fluxcd.io/flux/components/kustomize/kustomizations/#ignore-rules)
- [Kubernetes CronJob time zones](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#time-zones)
- [Excel-conversion ScaledJob patches](Flux/Tst/excel-conversion/patch.keda.scaledjob.yml)
