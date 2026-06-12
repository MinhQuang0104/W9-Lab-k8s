# W9 Lab - GitOps Canary, SLO va Auto-Rollback

Du an minh hoa quy trinh GitOps tren Kubernetes:

- Argo CD tu dong dong bo manifest tu Git.
- Argo Rollouts trien khai API theo chien luoc canary.
- Prometheus thu thap metric va danh gia chat luong ban moi.
- AnalysisRun tu dong abort canary khi SLO khong dat.
- Alertmanager gui email khi ti le loi vuot nguong.

## Kien truc

```text
GitHub
  -> Argo CD
  -> Argo Rollouts
  -> Service api
  -> API Pods
       -> /metrics
       -> ServiceMonitor
       -> Prometheus
            -> AnalysisRun -> abort/rollback
            -> PrometheusRule -> Alertmanager -> email
```

API Flask trong `app/app.py` doc hai bien moi truong:

```python
ERR = float(os.getenv("ERROR_RATE", "0"))
VER = os.getenv("VERSION", "v1")
```

Moi request vao `/` co xac suat tra HTTP 500 bang `ERROR_RATE`. Vi du,
`ERROR_RATE=0.5` tao xap xi 50% request loi tren cac pod cua ban canary.

## Cac manifest chinh

| File | Muc dich |
| --- | --- |
| `app/app.py` | API Flask va co che chen loi co chu dich. |
| `app/Dockerfile` | Dong goi API thanh image `w9-api:1`. |
| `k8s-api/api.yaml` | Rollout 4 replica, Service va cac buoc canary. |
| `k8s-api/analysis.yaml` | Query SLI va dieu kien chap nhan ban canary. |
| `k8s-api/servicemonitor.yaml` | Yeu cau Prometheus scrape `/metrics` moi 15 giay. |
| `k8s-api/prometheus-alerts.yaml` | Alert khi ti le HTTP 5xx vuot 2%. |
| `argocd/apps/api.yaml` | Argo CD Application quan ly workload API. |
| `argocd/apps/kube-prometheus-stack.yaml` | Cai Prometheus, Alertmanager va cau hinh email. |

## Chien luoc Canary

Rollout co 4 replica va cac buoc:

```yaml
steps:
  - setWeight: 25
  - pause: { duration: 1m }
  - setWeight: 50
  - pause: { duration: 1m }
  - setWeight: 100
```

Khi co revision moi, Rollout dua 25% traffic sang canary, cho mot phut, tang
len 50%, sau do moi co the dua len 100%. Background Analysis chay song song
voi cac buoc nay.

## SLI va nguong Analysis

Query trong `k8s-api/analysis.yaml`:

```promql
sum(rate(flask_http_request_total{status!~"5..", namespace="demo"}[2m]))
/
sum(rate(flask_http_request_total{namespace="demo"}[2m]))
```

Query tinh **success rate trong cua so 2 phut**:

```text
success rate = toc do request khong phai 5xx / toc do tat ca request
```

Dieu kien hien tai:

```yaml
interval: 30s
successCondition: result[0] >= 0.95
failureLimit: 5
```

- Prometheus duoc query moi 30 giay.
- Lan do dat khi success rate lon hon hoac bang 95%.
- Khi so lan that bai vuot `failureLimit`, AnalysisRun that bai va Rollout abort.
- Voi `failureLimit: 5`, qua trinh thuong abort sau lan that bai thu 6, khoang
  2.5-3 phut tuy thoi diem metric bat dau co du du lieu.

### Vi sao ban `v2-error` bi abort?

Tai buoc 25%, neu canary co `ERROR_RATE=0.5`:

```text
ti le loi toan he thong ~= 25% traffic canary x 50% loi
                         = 12.5%

success rate ~= 100% - 12.5% = 87.5%
```

`87.5% < 95%`, do do Analysis lien tuc that bai. Khi dat gioi han, Argo
Rollouts abort revision loi, scale ReplicaSet canary ve 0 va dua traffic ve
ReplicaSet stable `v1`.

Gia tri thuc te co the chenhlech do phan phoi traffic, scrape interval, health
check va request `/metrics`. Can duy tri load vao endpoint `/` trong suot demo.

## Alert SLO va email

Query alert trong `k8s-api/prometheus-alerts.yaml`:

```promql
sum(rate(flask_http_request_total{status=~"5..", namespace="demo"}[1m]))
/
sum(rate(flask_http_request_total{namespace="demo"}[1m])) > 0.02
```

Query tinh ti le HTTP 5xx trong mot phut. Alert `ApiFastBurnRate` chuyen sang
`FIRING` khi:

- Ti le loi lon hon 2%.
- Dieu kien ton tai lien tuc trong `for: 1m`.

Alertmanager co `group_wait: 30s`, vi vay email thuong den sau khi alert bat
dau firing khoang 30 giay. `failureLimit: 5` duoc chon de email co thoi gian
gui truoc khi Rollout auto-abort.

## Kich ban demo

### 1. Trang thai an toan ban dau

```yaml
- { name: ERROR_RATE, value: "0" }
- { name: VERSION, value: "v1" }
```

Theo doi Rollout:

```bash
kubectl argo rollouts get rollout api -n demo --watch
```

Tao load lien tuc:

```bash
kubectl run api-load -n demo --image=curlimages/curl --restart=Never \
  --command -- sh -c 'while true; do curl -s -o /dev/null http://api:8080/; sleep 0.05; done'
```

### 2. Push ban loi

Sua `k8s-api/api.yaml`:

```yaml
- { name: ERROR_RATE, value: "0.5" }
- { name: VERSION, value: "v2-error" }
```

```bash
git commit -am "deploy v2 with error rate 0.5"
git push
```

### 3. Ket qua mong doi

1. Argo CD tu dong sync revision moi.
2. Rollout tao ReplicaSet canary va dua 25% traffic sang ban loi.
3. AnalysisRun `api-success-rate-check` ghi nhan success rate duoi 95%.
4. Prometheus alert chuyen `Pending`, sau do `Firing`.
5. Alertmanager gui email canh bao.
6. AnalysisRun that bai va Rollout chuyen sang `Aborted`.
7. Canary bi scale ve 0; 100% traffic tro lai ReplicaSet stable.

### 4. Don dep sau demo

Khoi phuc `ERROR_RATE=0`, `VERSION=v1`, commit va push. Xoa pod tao load:

```bash
kubectl delete pod api-load -n demo
```

## Bang chung

Dat clip va anh chup that vao `docs/evidence/` theo huong dan trong
[`docs/evidence/README.md`](docs/evidence/README.md).

| Bang chung | Tep de xuat |
| --- | --- |
| Clip toan bo canary, Analysis failed va auto-abort | `docs/evidence/canary-auto-abort.mp4` |
| Rollout dang chay canary 25% | `docs/evidence/01-canary-25-percent.png` |
| AnalysisRun failed | `docs/evidence/02-analysis-failed.png` |
| Rollout Aborted va stable nhan lai traffic | `docs/evidence/03-auto-abort-rollback.png` |
| Prometheus alert o trang thai FIRING | `docs/evidence/04-alert-firing.png` |
| Email canh bao tu Alertmanager | `docs/evidence/05-alert-email.png` |

Khong dua Gmail App Password, Kubernetes Secret hoac token truy cap vao anh,
clip hay Git repository.

