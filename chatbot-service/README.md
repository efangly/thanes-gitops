# k8s manifests — lims-chatbot-service on OKE

Deploys the NestJS + LangGraph.js AI chatbot microservice (repo:
`lims-chatbot-service`, sibling to `backend/` on disk, separate git repo) -
see that repo's `README.md` for the application itself. This folder is
Phase 5.1 of `/Users/tng-mac-01/.claude/plans/ai-chatbot-groovy-spindle.md`.

Talks to `../mcp-server/` (internal-only) for LIMS data and to the
Anthropic API directly for the LLM. Does **not** talk to Postgres, Redis,
or Oracle - it has no database of its own (see the plan's Phase 4/6 notes:
conversation memory is an in-memory `MemorySaver` for now, no persistence).

## Before applying

- **`../mcp-server/` (and therefore `../backend/`) should be applied
  first**, or at least before this service is expected to actually answer
  questions - `MCP_SERVER_URL` below points at it. The chatbot-service pod
  itself boots fine without a reachable MCP server (nothing connects until
  a request hits `POST /chat` - see the app repo's README), so ordering
  here is about functionality, not a hard apply-time dependency.
- The `thanes-lims-mcp-service-api-key` OCI Vault secret must already exist
  (see `../mcp-server/README.md`) - this service's `ExternalSecret`
  (`01-external-secret.yaml`) reads the same vault key.
- `02-deployment.yaml`: `image` → bump to a real tag once the
  `lims-chatbot-service` repo has a release workflow and has pushed at
  least one image (see that repo's `.github/workflows/release.yml`, added
  alongside these manifests - it needs `DOCKER_USERNAME`/`DOCKER_PASSWORD`,
  `GITOPS_PAT`, and the Telegram secrets configured on that repo before a
  tag push will actually build/push/bump anything, same as `../backend/`).
- `04-gateway-httproute.yaml`: external path is `/ai` (rewritten to `/`
  before reaching the pod, so `/ai/chat` → the app's `POST /chat`) on the
  same hostname as everything else, `lims.siamatic.work`. This was chosen
  per the migration plan's Phase 5.1 as a placeholder - confirm the actual
  path with whoever owns the frontend before the real cutover, and update
  backend `docs/chatbot-frontend-integration.md` to match once confirmed
  (Phase 5.3 of the plan).
- No `Gateway`/`Certificate` object is defined here, same as `../backend/`
  - this shares the existing `lims-gateway` Gateway and `lims-tls`
  Certificate (`../platform/gateway.yaml`, `../platform/certificate.yaml`).

## Apply order

```sh
kubectl apply -f 00-configmap.yaml
kubectl apply -f 01-external-secret.yaml
kubectl apply -f 02-deployment.yaml
kubectl apply -f 03-service.yaml
kubectl apply -f 04-gateway-httproute.yaml
```

Or, once the vault secret exists:

```sh
kubectl apply -f chatbot-service/
```

## Verify

```sh
kubectl get pods -n thanes-lims -l app=lims-chatbot-service
kubectl get httproute -n thanes-lims lims-chatbot-service
curl https://lims.siamatic.work/ai/chat \
  -X POST -H "Content-Type: application/json" \
  -H "Authorization: Bearer <a real access token with chatbot:view>" \
  -d '{"question": "มี sample อะไรบ้างที่ยังค้างสถานะ pending เกิน 7 วัน?"}'
```

Run through the 6 scenarios in backend `docs/chatbot-acceptance-checklist.md`
this way as part of Phase 5.2's verification gate, before Phase 2 removes
the old Go `/api/v1/chat` route.

`kubectl apply --dry-run=client -f chatbot-service/` checks manifest syntax
without a live cluster.
