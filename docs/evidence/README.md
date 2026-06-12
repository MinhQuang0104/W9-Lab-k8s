# Evidence

Thu muc nay chua bang chung cho kich ban canary auto-abort va rollback.

## Danh sach can chup

1. `01-canary-25-percent.png`: Rollout co stable ReplicaSet va canary
   ReplicaSet, canary dang nhan 25% traffic.
2. `02-analysis-failed.png`: AnalysisRun `api-success-rate-check` hien cac lan
   do that bai hoac trang thai `Failed`.
3. `03-auto-abort-rollback.png`: Rollout hien `Aborted`, canary ve 0 replica va
   stable ReplicaSet nhan lai toan bo traffic.
4. `04-alert-firing.png`: Prometheus hien `ApiFastBurnRate` o trang thai
   `FIRING`.
5. `05-alert-email.png`: email canh bao da den hop thu; che thong tin nhay cam.
6. `canary-auto-abort.mp4`: clip lien tuc tu luc push revision loi den luc
   alert va auto-abort.

## Lenh ho tro quay

```bash
kubectl argo rollouts get rollout api -n demo --watch
kubectl get analysisrun -n demo --watch
kubectl get rs -n demo -l app=api --watch
```

Khong commit Gmail App Password, noi dung Secret, token hoac kubeconfig.
