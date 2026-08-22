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
├── backend/    # thanes-lims-backend API (namespace: thanes-lims)
├── frontend/   # thanes-lims frontend (namespace: thanes-lims)
└── platform/   # cluster-level resources ที่ทั้งระบบใช้ร่วมกัน
    └── argocd/ # ArgoCD server (namespace: argocd)
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

# 3. AppProject + Applications (ให้ ArgoCD ดูแล frontend/backend ต่อ)
kubectl apply -f platform/argocd/appproject-lims.yaml
kubectl apply -f platform/argocd/application-frontend.yaml
kubectl apply -f platform/argocd/application-backend.yaml
# ArgoCD จะ sync frontend/ และ backend/ ให้เองจากจุดนี้ (automated, prune, selfHeal ปิด)
# ต้องสร้าง Secret จริงก่อน sync backend สำเร็จ (ดู header ของ backend/01-secret.yaml)
```

หรือถ้าไม่ใช้ ArgoCD (apply ตรง ๆ):

```sh
# Frontend
kubectl apply -f frontend/

# Backend — ดู placeholder ที่ต้องแก้ก่อนใน backend/README.md
kubectl apply -f backend/00-namespace.yaml
# สร้าง Secret จริง (ดู header ของ backend/01-secret.yaml)
kubectl apply -f backend/02-configmap.yaml
kubectl apply -f backend/03-deployment.yaml
kubectl apply -f backend/04-service.yaml
kubectl apply -f backend/05-gateway-httproute.yaml
```

## ก่อน Apply — สิ่งที่ต้องแก้

- `platform/cluster-issuer.yaml`: `PLACEHOLDER_EMAIL` → อีเมลจริงสำหรับ Let's Encrypt
- `backend/03-deployment.yaml`: image → Docker Hub repo/tag จริง
- `backend/02-configmap.yaml`: `MINIO_ENDPOINT` → host:port ของ MinIO จริง
- `backend/01-secret.yaml`: เป็น template เท่านั้น ห้าม commit ค่าจริง —
  สร้าง Secret ตรง ๆ ด้วย `kubectl` หรือใช้เครื่องมือจัดการ secret
  (sealed-secrets, external-secrets ฯลฯ)

## Verify

```sh
kubectl get pods -n thanes-lims
kubectl get gateway,httproute -A
kubectl get certificate -A
curl https://lims.siamatic.work/
curl https://lims.siamatic.work/api/v1/health
curl https://argocd.siamatic.work/
```

`kubectl apply --dry-run=client -f <dir>/` ใช้เช็ค syntax ของ manifest
โดยไม่ต้องมีคลัสเตอร์จริง
</content>
