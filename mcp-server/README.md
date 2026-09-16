# k8s manifests — LIMS MCP server on OKE

Deploys `cmd/mcp-server` from the `backend` repo (Go, hexagonal) - a
standalone binary, separate from the main API, exposing typed read-only
Sample/TestResult/Inventory/PurchaseOrder tools over MCP Streamable HTTP for
`../chatbot-service` (the NestJS + LangGraph.js chatbot) to call. It reads
Postgres directly; no Oracle, no relation to the old chatbot POC being
retired from the API.

See backend `docs/mcp-server-tools.md` (the tool catalog / connection
contract) and `docs/adr/0013-mcp-server-transport.md` (transport choice)
for the application-level design. This folder is Phase 5.1 of
`/Users/tng-mac-01/.claude/plans/ai-chatbot-groovy-spindle.md`.

## Before applying

- **`../backend/` must be applied/synced first.** This Deployment has no
  ConfigMap/Secret of its own - it reuses `thanes-lims-config` and
  `thanes-lims-secrets` via `envFrom` (see `00-deployment.yaml`'s comments
  for why: `cmd/mcp-server` shares the same Go `Config` struct as `cmd/api`,
  so it needs the *entire* config, not just the two MCP-specific vars).
  Specifically it needs `../backend/02-configmap.yaml`'s `MCP_SERVER_PORT`
  and `../backend/06-external-secret.yaml`'s `MCP_SERVICE_API_KEY` to exist
  - both were added there, not here, for that reason.
- The `thanes-lims-mcp-service-api-key` OCI Vault secret (see
  `../backend/SECRETS-OCI-VAULT.md` for how the other vault secrets were
  provisioned - add this one the same way) must exist before
  `../backend/06-external-secret.yaml` can sync successfully. Generate a
  long random value (e.g. `openssl rand -hex 32`) - it's a bearer secret,
  not a password, and the exact same value must also be provisioned as
  `../chatbot-service`'s `MCP_SERVICE_API_KEY` (same vault key, both
  ExternalSecrets reference it - see `../chatbot-service/README.md`).
- `00-deployment.yaml`: `image` must match whatever tag
  `../backend/03-deployment.yaml` is pinned to - they're built from the same
  Dockerfile/release. Bumped automatically by the backend repo's
  `.github/workflows/release.yml` (same job that bumps `../backend/`) - see
  that repo's Dockerfile, which now builds both `./api` and `./mcp-server`
  into the same image. Release `0.0.19` (backend commit `7440b41`) was the
  first to include `./mcp-server` - don't pin this to anything older, the
  pod will `CrashLoopBackOff` on `exec ./mcp-server: no such file or
  directory`.
- No HTTPRoute exists for this Service anywhere in this repo, on purpose -
  it's internal-only (`ClusterIP`, no Gateway attachment). Don't add one.
- `02-networkpolicy.yaml` only takes effect if the cluster's CNI enforces
  `NetworkPolicy` (OKE's default Flannel CNI does not - see the file's
  header comment). Confirm enforcement is active, or treat this as
  documentation-of-intent rather than an enforced boundary.

## Apply order

```sh
kubectl apply -f ../backend/02-configmap.yaml
kubectl apply -f ../backend/06-external-secret.yaml
kubectl apply -f 00-deployment.yaml
kubectl apply -f 01-service.yaml
kubectl apply -f 02-networkpolicy.yaml
```

Or, once `../backend/`'s config/secret are already applied:

```sh
kubectl apply -f mcp-server/
```

## Verify

```sh
kubectl get pods -n thanes-lims -l app=lims-mcp-server
# No unauthenticated health route exists - a bare curl gets 401, which is
# expected and means the process is up:
kubectl run -n thanes-lims tmp-curl --rm -it --restart=Never --image=curlimages/curl -- \
  curl -sS -o /dev/null -w '%{http_code}\n' http://lims-mcp-server:8090/
# From inside chatbot-service's pod (or any pod, until the NetworkPolicy is
# confirmed enforced), the full auth'd call chatbot-service actually makes
# looks like:
#   curl -X POST http://lims-mcp-server.thanes-lims.svc.cluster.local:8090/ \
#     -H "X-Service-Api-Key: <MCP_SERVICE_API_KEY>" \
#     -H "Authorization: Bearer <a real user access token>" \
#     -H "Content-Type: application/json" \
#     -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

`kubectl apply --dry-run=client -f mcp-server/` checks manifest syntax
without a live cluster.
