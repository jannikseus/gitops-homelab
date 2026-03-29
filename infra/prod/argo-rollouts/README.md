# Argo Rollouts

This directory vendors the official Argo Rollouts installation manifest.

Source:
- official release manifest from Argo Rollouts GitHub releases

Pinned version:
- v1.7.0

Upgrade process:
1. download the new official install.yaml for the target version
2. store it under `upstream/`
3. update `kustomization.yaml`
4. review Argo CD diff before syncing