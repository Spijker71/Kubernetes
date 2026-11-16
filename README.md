# Kubernetes Course Notes

Reorganized notes from the OneNote export in [Kubernetes.mht](Kubernetes.mht). The material combines course references, personal reminders, and troubleshooting conversations.

## Reference

- [Kubernetes course notes](kubernetes-course-notes.md): workloads, configuration, networking, probes, storage, and AKS tooling.
- [kubectl command cheat sheet](kubectl-cheat-sheet.md): common inspection, rollout, configuration, KEDA, and storage commands.

## How-to guides

- [How to troubleshoot AKS workload identity](how-to-troubleshoot-aks-workload-identity.md): federated identity, Key Vault CSI mounts, secret synchronization, and TLS trust errors.
- [How to configure KEDA Service Bus autoscaling](how-to-configure-keda-servicebus-autoscaling.md): choose a KEDA resource, configure a queue trigger, and investigate scaler failures.
- [How to troubleshoot Kubernetes manifests and storage](how-to-troubleshoot-kubernetes-manifests-and-storage.md): Kustomize builds, Deployment YAML, and PV/PVC states.

## Additional resources

- [Kubeschool](https://kubeschool.portainer.io/): Kubernetes course portal.
- [Dev cluster login script](new%20notes/Inloggen%20in%20Dev%20cluster.ps1): Azure CLI and kubelogin commands for AKS credentials, followed by a k9s launch.
- [Production cluster login script](new%20notes/Inloggen%20in%20Prrod%20cluster.ps1): Azure CLI and kubelogin commands for production AKS credentials, followed by a k9s launch.

## Notes on accuracy

These are personal course notes, not a tested runbook for a particular cluster. Replace placeholders, verify the current Kubernetes, AKS, CSI driver, and KEDA versions, and confirm the active context before applying changes. The additional login scripts contain organization-specific subscription and cluster identifiers; verify them before use, especially for production. Examples in the reorganized notes use placeholders. The original export is unchanged.
