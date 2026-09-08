# k8s manifests — thanes-lims-backend on OKE

Plain manifests (no Kustomize/Helm) for deploying the API to Oracle
Kubernetes Engine. Postgres is an external service already running
outside the cluster; object storage is OCI Object Storage (S3-compatible
API) — nothing here deploys a database or a bucket.

## Before applying

Edit these placeholders:

- `03-deployment.yaml`: `image: docker.io/siamatic/lims-backend:latest` is
  bumped to a specific version tag automatically by the backend repo's
  `.github/workflows/release.yml` on every semver tag push (e.g. `0.0.1`)
  — don't edit the tag by hand. Remove `imagePullSecrets` if the repo is
  public.
- `02-configmap.yaml`: `STORAGE_ENDPOINT` / `STORAGE_REGION` / `STORAGE_BUCKET`
  → your OCI Object Storage namespace, region and bucket.
- `05-gateway-httproute.yaml` already points at the shared `lims-gateway`
  Gateway (defined in `../platform/gateway.yaml`), `sectionName: https`,
  hostname `lims.siamatic.work`, path prefix `/api/v1`. No
  `Gateway`/`Certificate` object is defined here — the API shares the
  frontend's existing TLS listener and cert (`lims-tls`, in
  `../platform/certificate.yaml`) on the same hostname, split by path.
- `thanes-lims-secrets` is **no longer hand-managed**. Its 8 values live in
  OCI Vault and the External Secrets Operator projects them into the
  cluster — see `SECRETS-OCI-VAULT.md` for the one-time setup (Vault
  secrets, IAM policy, ESO install) and `06-external-secret.yaml` for the
  `ClusterSecretStore` / `ExternalSecret` that ArgoCD manages. Keys:
  `DATABASE_URL`, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `REDIS_URL`,
  `ANTHROPIC_API_KEY`, `ORACLE_DSN`, and `STORAGE_ACCESS_KEY` /
  `STORAGE_SECRET_KEY` (OCI IAM Customer Secret Key — was
  `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY`).
- `01-secret-adb-wallet.yaml` is also a template only (no data). The
  `adb-wallet` Secret backs the Oracle ADB wallet volume mounted at
  `/app/wallet` in the API container (used by the chatbot feature).
  Create it from a **container-ready copy** of the unzipped ADB wallet
  directory — not your local dev wallet — see that file's header comment
  for why (`sqlnet.ora`'s `WALLET_LOCATION` must point at `/app/wallet`,
  not your dev machine's path).

## Apply order

```sh
kubectl apply -f 00-namespace.yaml
# one-time: OCI Vault + IAM + ESO install, then the adb-wallet Secret
# (see SECRETS-OCI-VAULT.md and 01-secret-adb-wallet.yaml headers), then:
kubectl apply -f 02-configmap.yaml
kubectl apply -f 06-external-secret.yaml   # ESO -> Secret/thanes-lims-secrets
kubectl apply -f 03-deployment.yaml
kubectl apply -f 04-service.yaml
kubectl apply -f 05-gateway-httproute.yaml
```

Or, once the one-time setup is done:

```sh
kubectl apply -f backend/
```

## Verify

```sh
kubectl get pods -n thanes-lims
kubectl get httproute,gateway -n thanes-lims
curl https://<domain>/api/v1/health
```

`kubectl apply --dry-run=client -f backend/` checks manifest syntax
without a live cluster. If `kubeconform`/`kubeval` is installed, use it
for schema validation too.
