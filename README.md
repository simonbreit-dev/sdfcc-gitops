# SDFCC Flux GitOps Repository

This repository defines the desired Kubernetes state for SDFCC. Application
source and release builds remain in `software-development-cloud-computing26`;
this repository consumes the published OCI Helm chart and pins the promoted
application release.

The current implementation prepares the existing local Colima/K3s cluster.
Production resources are only a suspended scaffold and must not be reconciled
as a production environment yet.

## Repository layout

```text
clusters/
├── dev/
│   ├── namespace.yaml
│   ├── infrastructure.yaml       # Flux Kustomization, initially suspended
│   ├── apps.yaml                 # Depends on infrastructure, initially suspended
│   ├── infrastructure/
│   │   ├── postgresql/
│   │   └── secrets/
│   ├── apps/
│   │   └── sdfcc-backend/
│   └── kustomization.yaml
└── prod/
    ├── apps/sdfcc-backend/       # Deliberately suspended
    └── kustomization.yaml
```

The dev root creates two Flux Kustomizations. Infrastructure is reconciled
first; the application Kustomization starts only after infrastructure is Ready.
Both are committed with `spec.suspend: true` so publishing an incomplete draft
cannot accidentally deploy placeholder tags or start PostgreSQL without its
Secret.

The local PostgreSQL image is pinned to the verified multi-platform OCI index
for `postgres:17.6-alpine`, which includes Linux/ARM64 for the Colima node.

## Artifact contract

The application CI publishes:

```text
image: ghcr.io/simonbreit-dev/software-development-cloud-computing26/sdfcc-backend:sha-<git-sha>
chart: ghcr.io/simonbreit-dev/software-development-cloud-computing26/charts/sdfcc-backend:0.1.0-sha-<git-sha>
```

Before unsuspending the dev application, replace both `REPLACE_ME` markers with
the matching tags from one green CI run. Confirm that the published image
manifest contains `linux/arm64`; the local K3s node is ARM64. Do not use `latest`
or `main` in GitOps.

The OCI source selects the packaged Helm layer explicitly for Flux
`HelmRelease.spec.chartRef` consumption.

## Database Secret with SOPS and age

Plaintext secrets and local age private keys are ignored by `.gitignore`.
Install the currently missing tools when preparing the first deployment:

```sh
brew install sops age
age-keygen -o .age-key.txt
age-keygen -y .age-key.txt
```

Create `.sops.yaml` using the printed public recipient:

```yaml
creation_rules:
  - path_regex: clusters/dev/infrastructure/secrets/.*\.secret\.sops\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1REPLACE_WITH_YOUR_PUBLIC_RECIPIENT
```

Copy the example, generate a unique password, and keep the plaintext file
uncommitted:

```sh
cp clusters/dev/infrastructure/secrets/sdfcc-database.secret.yaml.example \
  clusters/dev/infrastructure/secrets/sdfcc-database.secret.yaml
sops --encrypt \
  clusters/dev/infrastructure/secrets/sdfcc-database.secret.yaml \
  > clusters/dev/infrastructure/secrets/sdfcc-database.secret.sops.yaml
```

Add `sdfcc-database.secret.sops.yaml` to the resources in
`clusters/dev/infrastructure/secrets/kustomization.yaml`. Verify that the
encrypted file contains SOPS metadata and no readable password before staging
it.

Install the age private key in the cluster as an out-of-band bootstrap secret:

```sh
kubectl --namespace flux-system create secret generic sops-age \
  --from-file=age.agekey=.age-key.txt
```

Back up `.age-key.txt` in an appropriate password manager or secret store. Do
not commit it. Losing it makes the encrypted repository Secret unrecoverable.

## Publish and bootstrap

This local directory currently has no commit or remote. Create and publish the
GitHub repository before changing the cluster. The running Flux installation is
still connected to `simonbreit-dev/fsdfcc-gitops` at `./clusters/my-cluster`.

Once this repository is published and the dev tree is complete, re-bootstrap
the local cluster deliberately:

```sh
flux bootstrap github \
  --owner=simonbreit-dev \
  --repository=sdfcc-gitops \
  --branch=main \
  --path=clusters/dev \
  --personal
```

Bootstrap creates `clusters/dev/flux-system`. Because the dev root already has
a Kustomize file, ensure that `clusters/dev/kustomization.yaml` includes
`flux-system` after bootstrap, then commit and push that generated change. Check
the resulting live source before assuming the switch succeeded:

```sh
kubectl --namespace flux-system get gitrepository flux-system
kubectl --namespace flux-system get kustomization flux-system
```

## First local deployment

Complete these steps in order:

1. Increase Colima memory to at least 4 GiB; 6 GiB is preferable with
   observability workloads.
2. Publish a green multi-platform application release.
3. Replace `REPLACE_ME` in the dev chart and image tags.
4. Create and commit the SOPS-encrypted database Secret.
5. Install the age private key as `flux-system/sops-age`.
6. Publish and bootstrap this repository.
7. Set `spec.suspend: false` in `clusters/dev/infrastructure.yaml`, commit, push,
   and wait until PostgreSQL and its PVC are Ready.
8. Set `spec.suspend: false` in `clusters/dev/apps.yaml`, commit, and push.
9. Reconcile and verify the release.

Useful checks:

```sh
kubectl kustomize clusters/dev
kubectl kustomize clusters/dev/infrastructure
kubectl kustomize clusters/dev/apps
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
flux get kustomizations --all-namespaces
flux get sources all --all-namespaces
flux get helmreleases --all-namespaces
kubectl get pods,services,persistentvolumeclaims --namespace sdfcc-dev
kubectl get events --namespace sdfcc-dev --sort-by=.lastTimestamp
```

Access the backend without installing an ingress controller:

```sh
kubectl --namespace sdfcc-dev port-forward service/sdfcc-backend 8080:8080
curl --fail --silent http://127.0.0.1:8080/actuator/health/readiness
```

Then test at least one database-backed API operation. Readiness alone does not
validate the incomplete authentication flow documented in the application
repository's deployment audit.

## Updating configuration and releases

- Promote releases by changing the explicit chart and image SHA tags in a pull
  request.
- Values ConfigMaps carry `reconcile.fluxcd.io/watch: Enabled`, so Flux notices
  values changes promptly.
- The application chart checksums generated ConfigMaps and Secrets to roll pods
  when their contents change.
- For the externally managed SOPS Secret, increment `secret.rolloutToken` in
  `values.yaml` after changing database Secret data.
- Automatic upgrade remediation retries remain disabled while Hibernate uses
  `ddl-auto=update`; do not assume an application rollback is schema-safe.
- Keep the production OCI source and HelmRelease suspended until credentials,
  ingress/TLS, migrations, promotion, and production policies are designed.
