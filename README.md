# Software Development for Cloud Computing GitOps Repository

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

Dev currently selects chart `0.1.0-sha-4c16932` and image `sha-4c16932`. The
image is additionally pinned to multi-platform index digest
`sha256:0c482d7ff25e29e8689e7b020a1ef5b6ffd263a7165bf4da5aebc55869628730`,
which contains both `linux/amd64` and `linux/arm64`. The chart has matching
`appVersion: sha-4c16932`. Do not use `latest` or `main` in GitOps.

This release is selected for the local dev deployment with the current Trivy
findings accepted temporarily. Replace the chart tag, image tag, and image
digest together when promoting a newer release.

The OCI source selects the packaged Helm layer explicitly for Flux
`HelmRelease.spec.chartRef` consumption.

## Database Secret with SOPS and age

Plaintext secrets and local age private keys are ignored by `.gitignore`. Install
the tools if they are missing when preparing or rotating deployment secrets:

```sh
brew install sops age
age-keygen -o .age-key.txt
age-keygen -y .age-key.txt
```

The committed `.sops.yaml` contains the public recipient for this dev
environment. To rotate the age identity, replace its recipient with the output
of `age-keygen -y .age-key.txt`:

```yaml
creation_rules:
  - path_regex: clusters/dev/infrastructure/secrets/.*\.secret(\.sops)?\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1REPLACE_WITH_YOUR_PUBLIC_RECIPIENT
```

Copy the example, generate a unique database password and one persistent RSA
key pair, and keep the plaintext files uncommitted:

```sh
cp clusters/dev/infrastructure/secrets/sdfcc-database.secret.yaml.example \
  clusters/dev/infrastructure/secrets/sdfcc-database.secret.yaml
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 \
  -out .jwt-private.pem
openssl pkey -in .jwt-private.pem -pubout -out .jwt-public.pem
sops --encrypt \
  clusters/dev/infrastructure/secrets/sdfcc-database.secret.yaml \
  > clusters/dev/infrastructure/secrets/sdfcc-database.secret.sops.yaml
```

Replace every `CHANGE_ME` in the plaintext Secret before encryption. Insert the
complete private and public PEM contents as YAML block scalars under
`JWT_PRIVATE_KEY_PEM` and `JWT_PUBLIC_KEY_PEM`. The private key must be PKCS#8,
the public key must be X.509, and the pair must remain stable across pod
restarts. Delete `.jwt-private.pem`, `.jwt-public.pem`, and the plaintext Secret
after encryption and protected backup of the private key.

Add `sdfcc-database.secret.sops.yaml` to the resources in
`clusters/dev/infrastructure/secrets/kustomization.yaml`. Verify that the
encrypted file contains SOPS metadata and no readable password before staging
it.

For local decryption checks, point SOPS at the ignored repository key without
printing the Secret:

```sh
SOPS_AGE_KEY_FILE=.age-key.txt sops --decrypt \
  clusters/dev/infrastructure/secrets/sdfcc-database.secret.sops.yaml \
  >/dev/null
```

Install the age private key in the cluster as an out-of-band bootstrap secret:

```sh
kubectl --namespace flux-system create secret generic sops-age \
  --from-file=age.agekey=.age-key.txt
```

Back up `.age-key.txt` in an appropriate password manager or secret store. Do
not commit it. Losing it makes the encrypted repository Secret unrecoverable.

## Publish and bootstrap

This repository is published privately at
`github.com/simonbreit-dev/sdfcc-gitops`. Flux bootstrap resources have not yet
been committed here. Verify the live cluster source before changing it; the
last recorded installation still used `simonbreit-dev/fsdfcc-gitops` at
`./clusters/my-cluster`.

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

1. Run Colima with Kubernetes enabled and at least 4 GiB memory; 6 GiB is
   preferable with observability workloads.
2. Confirm the SOPS-encrypted database/JWT Secret is committed and securely back
   up the ignored `.age-key.txt` private key.
3. Install that age private key as `flux-system/sops-age`.
4. Bootstrap Flux from this repository.
5. Set `spec.suspend: false` in `clusters/dev/infrastructure.yaml`, commit, push,
   and wait until PostgreSQL and its PVC are Ready.
6. Set `spec.suspend: false` in `clusters/dev/apps.yaml`, commit, and push.
7. Reconcile and verify the release.

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
