# thanes-lims-secrets → OCI Vault (via External Secrets Operator)

Goal: stop hand-managing the `thanes-lims-secrets` Kubernetes Secret. The values
live in **OCI Vault** (`thanes-lims-backend-vault`) and the
**External Secrets Operator** (ESO) projects them into the same Secret the
Deployment already consumes (`envFrom.secretRef`).

```
OCI Vault secret  ──ESO (InstancePrincipal)──►  Secret/thanes-lims-secrets  ──►  api pod env
```

Manifests: `06-external-secret.yaml` (ArgoCD-managed). ESO operator itself is
installed once, out of band.

Identifiers used below:

| thing | value |
|---|---|
| Region | `ap-samutprakan-1` (realm OC43, domain `oci.thaiaiscloud.com`) |
| Compartment | `ocid1.compartment.oc43..aaaaaaaacykfvbprbe7kiwqs3ingrgumelfcsvuxfzdgg2uujlzn3ws5op6q` |
| Vault | `ocid1.vault.oc43.ap-samutprakan-1.rfvitu3faaa2k.abrgqljr6coo3adcn3ajyumrdhal77acv6ny2pjjs46vbejh5tro4a4ioybq` |
| OKE cluster | `tn-cluster` (BASIC_CLUSTER — no Workload Identity) |
| Worker nodes compartment | `thanes-lims` (same as above) |

---

## Step 1 — OCI Vault: master key + 8 secrets

Master encryption key — **already created** in `thanes-lims-backend-vault`:

```bash
COMPARTMENT=ocid1.compartment.oc43..aaaaaaaacykfvbprbe7kiwqs3ingrgumelfcsvuxfzdgg2uujlzn3ws5op6q
VAULT=ocid1.vault.oc43.ap-samutprakan-1.rfvitu3faaa2k.abrgqljr6coo3adcn3ajyumrdhal77acv6ny2pjjs46vbejh5tro4a4ioybq
KEY_ID=ocid1.key.oc43.ap-samutprakan-1.rfvitu3faaa2k.abrgqljr6i7lnyuvnuqncnjewiajhv2ovabk3hzbbtvmz2ehrfhkc5ctrpga  # thanes-lims-secrets-key (AES-256, SOFTWARE)
```

(If it ever needs recreating: `oci kms management key create --compartment-id "$COMPARTMENT"
--endpoint https://rfvitu3faaa2k-management.kms.ap-samutprakan-1.oci.thaiaiscloud.com
--display-name thanes-lims-secrets-key --key-shape '{"algorithm":"AES","length":32}'
--protection-mode SOFTWARE`.)

Then create one secret per value. Current values come from the live cluster
(base64-decode `kubectl get secret thanes-lims-secrets -n thanes-lims -o jsonpath='{.data.KEY}' | base64 -d`).
**`STORAGE_*` are new** — the OCI IAM user Customer Secret Key (see backend
`docs/adr/0009`), replacing `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY`.

```bash
mksecret() {  # $1 = secret name, $2 = plaintext value
  oci vault secret create-base64 \
    --compartment-id "$COMPARTMENT" --vault-id "$VAULT" --key-id "$KEY_ID" \
    --secret-name "$1" \
    --secret-content-content "$(printf %s "$2" | base64)"
}

mksecret thanes-lims-database-url        '<DATABASE_URL>'
mksecret thanes-lims-jwt-access-secret   '<JWT_ACCESS_SECRET>'
mksecret thanes-lims-jwt-refresh-secret  '<JWT_REFRESH_SECRET>'
mksecret thanes-lims-redis-url           '<REDIS_URL>'
mksecret thanes-lims-anthropic-api-key   '<ANTHROPIC_API_KEY>'
mksecret thanes-lims-oracle-dsn          '<ORACLE_DSN>'
mksecret thanes-lims-storage-access-key  '<Customer Secret Key: id>'
mksecret thanes-lims-storage-secret-key  '<Customer Secret Key: secret>'
```

Secret names must match `remoteRef.key` in `06-external-secret.yaml` exactly.

---

## Step 2 — IAM: let the worker nodes read the vault

Instance Principal auth. Create a dynamic group covering the OKE nodes and a
policy granting it read on the secret contents.

```bash
# Dynamic group (in the default identity domain)
oci iam dynamic-group create \
  --name thanes-lims-oke-nodes \
  --description "OKE worker nodes for thanes-lims" \
  --matching-rule "ALL {instance.compartment.id = '$COMPARTMENT'}"

# Policy on the compartment
oci iam policy create \
  --compartment-id "$COMPARTMENT" \
  --name thanes-lims-eso-vault-read \
  --description "ESO reads thanes-lims secrets from Vault" \
  --statements '["Allow dynamic-group thanes-lims-oke-nodes to read secret-family in compartment id '"$COMPARTMENT"'"]'
```

If this tenancy uses named identity domains, qualify the names
(`...dynamic-group '<domain>'/thanes-lims-oke-nodes ...`).

---

## Step 3 — Install External Secrets Operator (once)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace \
  --set installCRDs=true

kubectl -n external-secrets rollout status deploy/external-secrets
```

(Optional, matches the existing GitOps style: wrap this in an ArgoCD
`Application` pointing at the helm chart instead of `helm install`.)

---

## Step 4 — Cut over

```bash
# Back up the current hand-made secret, just in case
kubectl get secret thanes-lims-secrets -n thanes-lims -o yaml > /tmp/thanes-lims-secrets.bak.yaml

# Delete it so ESO can recreate it as Owner
kubectl delete secret thanes-lims-secrets -n thanes-lims

# Apply the ESO manifests (or let ArgoCD sync backend/)
kubectl apply -f 06-external-secret.yaml

# Watch it populate
kubectl get externalsecret thanes-lims-secrets -n thanes-lims -w
#   STATUS should become  SecretSynced / Ready=True
kubectl get secret thanes-lims-secrets -n thanes-lims -o go-template='{{range $k,$_ := .data}}{{$k}}{{"\n"}}{{end}}'
#   -> 8 keys incl. STORAGE_ACCESS_KEY / STORAGE_SECRET_KEY

# Roll the API so it picks up the (possibly changed) values
kubectl rollout restart deploy/thanes-lims-api -n thanes-lims
kubectl rollout status  deploy/thanes-lims-api -n thanes-lims
```

Verify: `curl https://lims.siamatic.work/api/v1/health`, then upload/download a
document (exercises the new `STORAGE_*` creds against OCI Object Storage).

---

## Rollback

`kubectl apply -f /tmp/thanes-lims-secrets.bak.yaml` recreates the old secret;
delete the `ExternalSecret` (or set ArgoCD to ignore) so ESO stops reconciling,
then `rollout restart`. Note the backup still has `MINIO_*` keys — fine only if
you also revert the backend image/config.

## Ongoing

- Rotate a value: update it in OCI Vault (new secret version). ESO re-syncs
  within `refreshInterval` (1h); force it with
  `kubectl annotate externalsecret thanes-lims-secrets -n thanes-lims force-sync=$(date +%s) --overwrite`.
  A changed Secret does **not** auto-restart pods — `rollout restart` after.
- New key: add a secret in Vault + a `data:` entry in `06-external-secret.yaml`.
