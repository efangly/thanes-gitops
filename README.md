# thanes-lims k8s manifests

Plain Kubernetes manifests (ไม่ใช้ Kustomize/Helm) สำหรับ deploy ระบบ
**thanes-lims** (frontend + backend API) และ platform resources ที่เกี่ยวข้อง
ขึ้น Oracle Kubernetes Engine (OKE) โดยใช้ Gateway API (Envoy Gateway) +
cert-manager สำหรับ TLS และ ArgoCD สำหรับ GitOps

Postgres และ MinIO เป็น external services ที่รันอยู่นอกคลัสเตอร์แล้ว
(บน OCI VM) — ไม่มีอะไรในนี้ deploy database

## โครงสร้างโปรเจค

```
.
├── backend/           # thanes-lims-backend API (namespace: thanes-lims)
├── frontend/          # thanes-lims frontend (namespace: thanes-lims)
├── mcp-server/        # Go MCP server (cmd/mcp-server, internal-only, namespace: thanes-lims)
├── chatbot-service/   # NestJS + LangGraph.js AI chatbot (namespace: thanes-lims)
└── platform/          # cluster-level resources ที่ทั้งระบบใช้ร่วมกัน
    └── argocd/        # ArgoCD server (namespace: argocd)
```

### `platform/`

Resource ระดับคลัสเตอร์ที่ต้อง apply ก่อน และใช้ร่วมกันระหว่าง frontend/backend/ArgoCD:

- `gatewayclass-envoy.yaml` — GatewayClass สำหรับ Envoy Gateway controller
- `gateway.yaml` — Gateway กลาง (`lims-gateway`) เปิด listener `http` (80),
  `https` สำหรับ `lims.siamatic.work` (443) และ `https-argocd` สำหรับ
  `argocd.siamatic.work` (443) พร้อม OCI Load Balancer annotations
- `cluster-issuer.yaml` — `ClusterIssuer` (Let's Encrypt prod) ใช้ออกใบรับรอง
  ผ่าน HTTP-01 challenge บน Gateway API
- `certificate.yaml` — ใบรับรอง TLS สำหรับ `lims.siamatic.work`
  (ใช้ร่วมกันทั้ง frontend และ backend เพราะ path ต่างกัน)
- `argocd/` — namespace, Certificate, HTTPRoute (+ redirect), ReferenceGrant
  และ config patch สำหรับติดตั้ง ArgoCD บนโดเมน `argocd.siamatic.work`

### `frontend/`

Deployment, Service, และ HTTPRoute (+ HTTPS redirect) ของหน้าเว็บ thanes-lims
เสิร์ฟบน path `/` ของ `lims.siamatic.work`

### `backend/`

Namespace, Secret (template), ConfigMap, Deployment, Service และ
Gateway HTTPRoute ของ API เสิร์ฟบน path prefix `/api/v1` ของโดเมนเดียวกัน
(ใช้ TLS cert ร่วมกับ frontend) ดูรายละเอียดการ deploy และ placeholder
ที่ต้องแก้ก่อน apply ได้ใน [`backend/README.md`](backend/README.md)

### `mcp-server/`

Deployment + Service + NetworkPolicy ของ Go MCP server (`cmd/mcp-server`
ใน repo `backend` เดียวกัน คนละ binary/entrypoint จาก API) เปิด tool
อ่านข้อมูล LIMS แบบ read-only ให้ `chatbot-service/` เรียกผ่าน MCP
Streamable HTTP — **internal-only ไม่มี HTTPRoute** ใช้ ConfigMap/Secret
ร่วมกับ `backend/` (ดู [`mcp-server/README.md`](mcp-server/README.md))
เป็นส่วนหนึ่งของแผนย้าย AI chatbot ออกจาก Go backend ไปเป็น service แยก
(`/Users/tng-mac-01/.claude/plans/ai-chatbot-groovy-spindle.md`)

### `chatbot-service/`

ConfigMap, Secret, Deployment, Service และ Gateway HTTPRoute ของ AI
chatbot microservice ใหม่ (NestJS + LangGraph.js, repo แยกต่างหาก
`lims-chatbot-service`) เสิร์ฟบน path prefix `/ai` ของโดเมนเดียวกัน
(rewrite `/ai/chat` → `/chat` ก่อนส่งเข้า pod) ดูรายละเอียดได้ใน
[`chatbot-service/README.md`](chatbot-service/README.md)

## ลำดับการ Apply

```sh
# 1. Platform (cluster-level) resources ก่อน
kubectl apply -f platform/gatewayclass-envoy.yaml
kubectl apply -f platform/gateway.yaml
kubectl apply -f platform/cluster-issuer.yaml   # แก้ email ใน cluster-issuer.yaml ก่อน
kubectl apply -f platform/certificate.yaml

# 2. ArgoCD (ถ้าต้องใช้ GitOps)
kubectl apply -f platform/argocd/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -n argocd
kubectl apply -f platform/argocd/cmd-params-patch.yaml
kubectl rollout restart deployment argocd-server -n argocd
kubectl apply -f platform/argocd/certificate.yaml
kubectl apply -f platform/argocd/referencegrant.yaml
kubectl apply -f platform/argocd/httproute.yaml
kubectl apply -f platform/argocd/httproute-redirect.yaml

# 3. AppProject + Applications (ให้ ArgoCD ดูแล frontend/backend/mcp-server/chatbot-service ต่อ)
kubectl apply -f platform/argocd/appproject-lims.yaml
kubectl apply -f platform/argocd/application-frontend.yaml
kubectl apply -f platform/argocd/application-backend.yaml
kubectl apply -f platform/argocd/application-mcp-server.yaml       # ต้องมี backend/ พร้อม MCP_SERVICE_API_KEY ใน vault แล้ว
kubectl apply -f platform/argocd/application-chatbot-service.yaml  # ต้องมี mcp-server/ พร้อมแล้ว
# ArgoCD จะ sync ให้เองจากจุดนี้ (automated, prune, selfHeal ปิด)
# ต้องสร้าง Secret จริงใน OCI Vault ก่อน sync backend/mcp-server/chatbot-service สำเร็จ
# (ดู backend/SECRETS-OCI-VAULT.md, backend/06-external-secret.yaml, chatbot-service/01-external-secret.yaml)
```

หรือถ้าไม่ใช้ ArgoCD (apply ตรง ๆ):

```sh
# Frontend
kubectl apply -f frontend/

# Backend — ดู placeholder ที่ต้องแก้ก่อนใน backend/README.md
kubectl apply -f backend/00-namespace.yaml
kubectl apply -f backend/02-configmap.yaml
kubectl apply -f backend/06-external-secret.yaml   # ต้องมี Secret จริงใน OCI Vault ก่อน
kubectl apply -f backend/03-deployment.yaml
kubectl apply -f backend/04-service.yaml
kubectl apply -f backend/05-gateway-httproute.yaml

# MCP server — ต้องมี backend/ พร้อมแล้ว (ใช้ ConfigMap/Secret ร่วมกัน)
kubectl apply -f mcp-server/

# Chatbot service — ต้องมี mcp-server/ พร้อมแล้ว
kubectl apply -f chatbot-service/
```

## ก่อน Apply — สิ่งที่ต้องแก้

- `platform/cluster-issuer.yaml`: `PLACEHOLDER_EMAIL` → อีเมลจริงสำหรับ Let's Encrypt
- `backend/03-deployment.yaml`: image → Docker Hub repo/tag จริง (bump อัตโนมัติจาก CI)
- `mcp-server/00-deployment.yaml`: image เป็น **placeholder** อยู่ (`PLACEHOLDER_NEEDS_RELEASE_AFTER_7440b41`) — ต้อง tag+release backend repo อีกครั้ง**หลัง**commit `7440b41` (ที่เพิ่ม `cmd/mcp-server` เข้า Dockerfile) ก่อน ถึงจะมี image ที่มี `./mcp-server` binary จริงให้ใช้ — อย่า apply manifest นี้ก่อนหน้านั้น (จะ CrashLoopBackOff)
- `chatbot-service/02-deployment.yaml`: image → Docker Hub repo/tag จริง (bump อัตโนมัติจาก CI ของ repo `lims-chatbot-service`)
- Secret ทั้งหมด (`thanes-lims-secrets`, `lims-chatbot-service-secrets`) ไม่ hand-manage แล้ว —
  มาจาก OCI Vault ผ่าน External Secrets Operator ดู `backend/SECRETS-OCI-VAULT.md`
  สำหรับ setup ครั้งแรก และต้องเพิ่ม vault key ใหม่ `thanes-lims-mcp-service-api-key`
  ก่อน apply `backend/06-external-secret.yaml`/`chatbot-service/01-external-secret.yaml`

## Verify

```sh
kubectl get pods -n thanes-lims
kubectl get gateway,httproute -A
kubectl get certificate -A
curl https://lims.siamatic.work/
curl https://lims.siamatic.work/api/v1/health
curl https://lims.siamatic.work/ai/chat -X POST -H "Content-Type: application/json" \
  -H "Authorization: Bearer <access token>" -d '{"question": "..."}'
curl https://argocd.siamatic.work/
```

`kubectl apply --dry-run=client -f <dir>/` ใช้เช็ค syntax ของ manifest
โดยไม่ต้องมีคลัสเตอร์จริง
</content>
