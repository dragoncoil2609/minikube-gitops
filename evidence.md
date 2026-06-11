# Báo Cáo Triển Khai: Argo Rollouts, Canary & SLO (Ship Smartly)

Tài liệu này trình bày giải pháp và kịch bản chi tiết để hoàn thành các Lab còn thiếu và thử thách "Ship Smartly". Quá trình triển khai sẽ tuân thủ tuyệt đối nguyên tắc GitOps (mọi thay đổi thông qua Git và Argo CD).

---

## 1. Lab 1 (Bổ sung): Cài đặt Argo Rollouts qua GitOps

**Mục tiêu:** Cài đặt Argo Rollouts Controller để k8s có khả năng hiểu được tài nguyên `Rollout` (thay vì `Deployment` truyền thống) và hỗ trợ chiến thuật Canary.

**Cách triển khai:**
Tạo file `argocd/apps/argo-rollouts.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-rollouts
  namespace: argocd
spec:
  source:
    repoURL: https://argo-rollouts.github.io/argo-rollouts
    chart: argo-rollouts
    targetRevision: 2.37.7
  destination:
    server: https://kubernetes.kubernetes.svc
    namespace: argo-rollouts
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```
**Lệnh áp dụng:** `git add argocd/apps/argo-rollouts.yaml && git commit -m "Add argo-rollouts" && git push`

---

## 2. Lab 2 & 3: Đưa ứng dụng vào k8s và cấu hình quét Metric

**Mục tiêu:** Build ứng dụng Python Flask (`w9-api:1`) có sẵn thư viện nhả metric tại `/metrics`, chuyển đổi nó thành đối tượng `Rollout` thay vì `Deployment`, và cấu hình `ServiceMonitor` để Prometheus thu thập data.

**Cách triển khai:**
1. Tạo thư mục `k8s-api/` chứa file `api.yaml` (Rollout + Service):
   - Chuyển `kind: Deployment` thành `kind: Rollout`.
   - Cấu hình chiến lược Canary thủ công (`pause: {}`) chờ con người quyết định:
     ```yaml
     strategy:
       canary:
         steps:
           - setWeight: 25
           - pause: {} # Dừng ở 25% chờ gõ lệnh promote
           - setWeight: 50
           - pause: { duration: 30s }
           - setWeight: 100
     ```
2. Tạo file `k8s-api/servicemonitor.yaml` để Prometheus nhắm vào port 8080 đường dẫn `/metrics`.
3. Tạo Application cho Argo CD: `argocd/apps/api.yaml` trỏ vào thư mục `k8s-api/`.
**Lệnh áp dụng:** Commit lên Git. Argo CD sẽ tự kéo và apply.

---

## 3. Lab 4: Chạy thử Canary bằng tay

**Kịch bản:** Sửa biến môi trường `VERSION` trong `k8s-api/api.yaml` từ `v1` sang `v2` và push lên Git.

**Quá trình xử lý:**
1. Argo CD tự động Sync, cập nhật Rollout.
2. Rollout sẽ tạo ra tập hợp Pods mới (v2) nhưng chỉ chuyển 25% lượng traffic vào v2, giữ nguyên 75% ở v1. Rollout chuyển sang trạng thái `Paused`.
3. Người vận hành theo dõi metric, nếu thấy ứng dụng ổn định sẽ gõ lệnh:
   ```bash
   kubectl argo rollouts promote api -n demo
   ```
   Nếu thấy lỗi (ví dụ 500 error tăng), gõ lệnh:
   ```bash
   kubectl argo rollouts abort api -n demo
   ```

---

## 4. Challenge: Ship Smartly (Tự động hóa hoàn toàn)

**Mục tiêu:** Loại bỏ yếu tố con người. Máy sẽ tự chấm điểm phiên bản mới (v2) thông qua Prometheus. Nếu phát hiện tỷ lệ lỗi > mức cho phép, máy tự `abort` và rollback về v1. Thêm cảnh báo gửi về Email khi lỗi.

**Cách triển khai:**

### 4.1. Auto-Canary với AnalysisTemplate
Tạo file `k8s-api/analysis.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: demo
spec:
  metrics:
  - name: success-rate
    interval: 20s
    successCondition: result[0] >= 0.95 # Phải đạt 95% tỷ lệ HTTP 200
    failureLimit: 2 # Cho phép xịt 2 lần trước khi abort
    provider:
      prometheus:
        address: http://monitoring-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
        query: |
          sum(rate(flask_http_request_total{status="200"}[1m])) 
          / 
          sum(rate(flask_http_request_total[1m]))
```
Chỉnh sửa lại phần `strategy.canary.steps` trong `Rollout`:
```yaml
      steps:
      - setWeight: 25
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50
      - pause: { duration: 1m }
      - setWeight: 100
```
**Giải thích:** Khi đạt mức 25%, Rollout sẽ tự kích hoạt `AnalysisTemplate` để gọi Prometheus. Prometheus trả về tỷ lệ thành công. Nếu `< 95%` quá 2 lần, Rollout tự động "đá" bản v2 đi và giữ lại v1.

### 4.2. Cảnh báo về Email (Alertmanager)
Thêm file cấu hình (hoặc sửa `app-monitoring.yaml`) để thêm PrometheusRule giám sát lỗi 500 và AlertManager config bắn qua SMTP (Gmail).
```yaml
# Mẫu Prometheus Rule
- alert: HighErrorRate
  expr: (sum(rate(flask_http_request_total{status="500"}[1m])) / sum(rate(flask_http_request_total[1m]))) > 0.05
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "High Error Rate Detected"
```

---
**Tổng kết:** Các bước trên đáp ứng đầy đủ yêu cầu: Triển khai 100% qua GitOps, Canary tự động đo lường bằng Prometheus, và tự động Rollback (Abort) nếu có lỗi.
