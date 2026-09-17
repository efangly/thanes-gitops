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
- `thanes-lims-secrets` is **no longer hand-managed**. Its values live in
  OCI Vault and the External Secrets Operator projects them into the
  cluster — see `SECRETS-OCI-VAULT.md` for the one-time setup (Vault
  secrets, IAM policy, ESO install) and `06-external-secret.yaml` for the
  `ClusterSecretStore` / `ExternalSecret` that ArgoCD manages. Keys:
  `DATABASE_URL`, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `REDIS_URL`,
  `STORAGE_ACCESS_KEY` / `STORAGE_SECRET_KEY` (OCI IAM Customer Secret Key
  — was `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY`), `PARTNER_API_KEY`
  (SMtrack Partner API gRPC key — see backend
  `docs/adr/0012-partner-api-grpc-transport.md`), and
  `MCP_SERVICE_API_KEY` (shared with `../mcp-server/` and
  `../chatbot-service/` — see those folders' READMEs). `ANTHROPIC_API_KEY`
  and `ORACLE_DSN` were removed here once the old in-repo chatbot POC was
  decommissioned (see backend `CONTEXT.md` "AI Chatbot") —
  `ANTHROPIC_API_KEY` now lives only in `../chatbot-service/`'s own
  ExternalSecret, and `ORACLE_DSN` isn't needed anywhere anymore (the
  `thanes-lims-oracle-dsn` Vault secret itself was left alone, just
  unreferenced — delete it in Vault too if you want to fully retire it).
- This Secret and ConfigMap are also consumed by `../mcp-server/`'s
  Deployment (`envFrom`, same resource names) — it shares this repo's Go
  `Config` struct (`internal/config/config.go`) and therefore needs every
  var here, not just the MCP-specific ones. Apply/sync this folder's
  `02-configmap.yaml`/`06-external-secret.yaml` before `../mcp-server/`.
- The `adb-wallet` Secret/volume (Oracle ADB wallet, used only by the old
  chatbot POC) has been removed from `03-deployment.yaml` — the container
  no longer needs cgo/Oracle Instant Client at all (see backend's
  Dockerfile). If an `adb-wallet` Secret object still exists in the
  cluster from before, it's now unused and safe to `kubectl delete secret
  adb-wallet -n thanes-lims` whenever convenient.

## Apply order

```sh
kubectl apply -f 00-namespace.yaml
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
