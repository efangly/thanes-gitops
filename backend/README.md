# k8s manifests — thanes-lims-backend on OKE

Plain manifests (no Kustomize/Helm) for deploying the API to Oracle
Kubernetes Engine. Postgres and MinIO are external services already
running outside the cluster — nothing here deploys a database.

## Before applying

Edit these placeholders:

- `03-deployment.yaml`: `image: docker.io/siamatic/lims-backend:latest` is
  bumped to a specific version tag automatically by the backend repo's
  `.github/workflows/release.yml` on every semver tag push (e.g. `0.0.1`)
  — don't edit the tag by hand. Remove `imagePullSecrets` if the repo is
  public.
- `02-configmap.yaml`: `MINIO_ENDPOINT` → your real MinIO host:port.
- `05-gateway-httproute.yaml` already points at the shared `lims-gateway`
  Gateway (defined in `../platform/gateway.yaml`), `sectionName: https`,
  hostname `lims.siamatic.work`, path prefix `/api/v1`. No
  `Gateway`/`Certificate` object is defined here — the API shares the
  frontend's existing TLS listener and cert (`lims-tls`, in
  `../platform/certificate.yaml`) on the same hostname, split by path.
- `01-secret.yaml` is a template only — don't edit and commit it. Create
  the real secret directly (see the command in that file's header
  comment) or manage it with your secrets tooling.

## Apply order

```sh
kubectl apply -f 00-namespace.yaml
# create the real Secret here (see 01-secret.yaml header), then:
kubectl apply -f 02-configmap.yaml
kubectl apply -f 03-deployment.yaml
kubectl apply -f 04-service.yaml
kubectl apply -f 05-gateway-httproute.yaml
```

Or, once placeholders are filled in and the real secret exists:

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
