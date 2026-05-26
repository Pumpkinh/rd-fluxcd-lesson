# rd-fluxcd-lesson

GitOps repository for Flux CD lessons.

## Structure

```text
base/course-app              # reusable course-app manifests
overlays/development         # development environment
overlays/production          # production environment
infrastructure/storage       # local PVs for the lab kubeadm cluster
clusters/k8scontrol          # Flux sync entry point
```

The cluster is bootstrapped by `clusters/k8scontrol/flux-instance.yaml`.
Flux then applies:

- `infra-storage`
- `app-dev`
- `app-prod`

Both environments use CloudNativePG (`kind: Cluster`) because the course app is PostgreSQL-based.
