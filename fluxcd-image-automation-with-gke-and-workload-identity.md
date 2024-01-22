---
tags: fluxcd,gcp,gitops,iam
---

# FluxCD image automation in GKE with Workload Identity

When using FluxCD inside GKE with Workload Identity, and storing images in Artifact Registry, there are a couple of ways to have FluxCD authenticate towards Artifact Registry. The more pragmatic way is to create a docker secret in the `flux-system` namespace. This method however requires you to create an IAM key and if your organisation is setup to rotate keys, that secret will eventually expire.

A better way is to setup an IAM account that maps to a service account in the `flux-system` namespace and use that mapping only for granting FluxCD access to Artifact Registry. Here is a minimal set of instructions to acheive this.

## Assumptions

- Flux is setup inside the namespace `flux-system`

## Steps

1. Create an IAM service account with `Artifact Registry Reader` role.
2. Create a mapping between the IAM SA from the previous step and the `image-reflector-controller` service account in the `flux-system` namespace.
3. Add a patch to the cluster's `kustomization.yaml`:

```yaml
patches:
  - patch: |
      apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: image-reflector-controller
        annotations:
          iam.gke.io/gcp-service-account: <iam-sa-email>
    target:
      kind: ServiceAccount
      name: image-reflector-controller
```

4. Configure Image Repositories to use the `image-reflector-controller` service account:

```yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: some-api
  namespace: flux-system
spec:
  image: <artifact-registry-image>
  interval: 30s
  provider: gcp
  serviceAccountName: image-reflector-controller
```
