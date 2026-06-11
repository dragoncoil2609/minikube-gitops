# Báo Cáo Triển Khai: GitOps & Ship Smartly (ArgoCD, Rollouts, Canary & SLO)

Tài liệu này tổng hợp toàn bộ quá trình thực hành từ con số 0 đến một hệ thống CI/CD & Observability hoàn chỉnh, tuân thủ tuyệt đối nguyên tắc GitOps.

---

## PHẦN 1: NỀN TẢNG GITOPS (BUỔI SÁNG)

### 1. Dựng cụm và khởi tạo Git (Lab 0)
- Khởi tạo cụm Kubernetes local bằng Minikube: `minikube start -p w9 --driver=docker`.
- Tạo repository `minikube-gitops` trên GitHub làm "Nguồn sự thật duy nhất" (Single Source of Truth).

### 2. Cài đặt ArgoCD (Lab 1)
- Cài đặt "người thợ" ArgoCD trực tiếp vào cụm qua lệnh `kubectl apply --server-side -n argocd -f <link_cài_đặt>`.
- Port-forward và lấy mật khẩu khởi tạo để truy cập vào UI của ArgoCD.

### 3. Application, Sync & Self-Heal (Lab 2 & 3)
- Tạo định nghĩa `Application` cho ứng dụng (ví dụ: `reactsurvey`). Lần đầu tiên, ta apply bằng tay lệnh `kubectl apply -f argocd/apps/reactsurvey.yaml`.
- **Sync:** Cập nhật file manifest trên Git (ví dụ tăng replicas), ArgoCD sẽ tự động nhận diện và đồng bộ trạng thái xuống cụm k8s.
- **Self-heal:** Thử nghiệm sửa thủ công trên cụm (scale tay replicas lên 9), ArgoCD ngay lập tức phát hiện lệch pha (OutOfSync) và tự động "chữa lành", ép số lượng Pod về lại đúng với cấu hình trên Git.

### 4. Rollback chuẩn GitOps (Lab 4)
- **Rollback chuẩn:** Thực hiện lệnh `git revert HEAD` để khôi phục code trên Git về trạng thái ổn định. ArgoCD sẽ tự động Sync cụm về theo.
- Không sử dụng `kubectl rollout undo` vì thao tác này làm sai lệch trạng thái thực tế so với Git, và ArgoCD sẽ tiếp tục "Self-heal" đè lên.

### 5. Mô hình App-of-Apps (Lab 5)
- Thay vì apply tay từng app, ta tạo một "Root Application" (`argocd/root.yaml`) quản lý toàn bộ thư mục `argocd/apps/`.
- Áp dụng lệnh `kubectl apply -f argocd/root.yaml` **lần cuối cùng**.
- Từ thời điểm này, để thêm app mới (như Prometheus, Rollouts), ta chỉ cần thả file cấu hình vào thư mục `argocd/apps/` và Push lên Git. Root App sẽ tự động sinh ra các App con.

### 6. Thứ tự triển khai với Sync Waves (Lab 6)
- Xử lý tình trạng ứng dụng bị lỗi do khởi tạo sai thứ tự bằng cách thêm Annotation `argocd.argoproj.io/sync-wave`.
- Ví dụ luồng triển khai được ép thứ tự: `Namespace (wave -1)` -> `Secret/ConfigMap (wave 0)` -> `Deployment Backend/Frontend/DB (wave 1)` -> `Service (wave 2)`.

### 7. Continuous Integration - CI (Lab 7)
- Thiết lập GitHub Actions (`.github/workflows/validate.yml`) chạy mỗi khi có Pull Request.
- Sử dụng công cụ `kubeconform` để rà soát (validate) các file YAML. Nếu file sai cấu trúc, quy tắc Branch Protection trên GitHub sẽ chặn nút Merge, bảo vệ nhánh `main` không bị nhiễm cấu hình rác.

---

## PHẦN 2: OBSERVABILITY, CANARY & ALERTING (BUỔI CHIỀU)

### 8. Cài đặt Argo Rollouts & Prometheus qua App-of-Apps
- Thêm file `argo-rollouts.yaml` và `app-monitoring.yaml` (dùng Helm chart của `kube-prometheus-stack`) vào `argocd/apps/` rồi Push lên Git. Root App tự động cài đặt 2 hệ thống này vào cụm k8s.

### 9. Triển khai Canary Deployment thủ công
- Chuyển đổi ứng dụng API Flask từ `Deployment` truyền thống sang đối tượng `Rollout`.
- Viết chiến lược Canary thủ công với các bước `setWeight: 25`, sau đó `pause: {}`. Hệ thống sẽ chuyển 25% traffic vào bản mới và chờ kỹ sư kiểm tra rồi gõ lệnh `kubectl argo rollouts promote` (đi tiếp) hoặc `abort` (quay xe) bằng tay.

### 10. Auto-Canary (Tự động hóa hoàn toàn với AnalysisTemplate)
- **Mục tiêu:** Loại bỏ sự can thiệp của con người.
- **Thực hiện:**
  - Viết `AnalysisTemplate` truy vấn Prometheus (đo tỷ lệ HTTP 200). Đặt thuộc tính `count` (ví dụ `count: 6`, mỗi lần 10s) để đo trong 1 khoảng thời gian cố định. Điều kiện pass là `result[0] >= 0.90`.
  - Tích hợp Analysis vào bước Pause của Rollout.
- **Kết quả:** Khi tung ra phiên bản lỗi (`ERROR_RATE=0.5`), Analysis phát hiện tỷ lệ thành công bị rớt xuống dưới 90%. Rollout ngay lập tức bị giật còi `Failed`, trạng thái chuyển sang `Degraded`, và hệ thống **tự động Abort**, tiêu diệt Pod lỗi và khôi phục 100% traffic về phiên bản cũ an toàn.

### 11. Cảnh báo lỗi tự động qua Email (Alertmanager)
- Kích hoạt Alertmanager trong file `app-monitoring.yaml`. Cấu hình SMTP trỏ về Gmail người nhận.
- Tránh lộ mật khẩu trên Git bằng cách tạo Kubernetes Secret `alertmanager-secret` chứa **Google App Password** (do chính người dùng tạo qua lệnh `kubectl` trực tiếp).
- Viết `PrometheusRule` quy định: "Nếu tỷ lệ lỗi HTTP 500 của API vượt quá 10% trong vòng 1 phút, kích hoạt nhãn HighErrorRate".
- **Kết quả:** Khi API bị sập hoặc gặp lỗi diện rộng, Alertmanager sẽ nhận tín hiệu và gửi ngay một email cảnh báo khẩn cấp tới hòm thư của đội ngũ DevOps.

---
**TỔNG KẾT:** Hệ thống đã hoàn thiện toàn bộ vòng đời của một ứng dụng Cloud Native hiện đại. Từ khâu tự động kiểm tra code (CI), tự động triển khai (CD - GitOps), phát hành an toàn từng phần (Canary), tự chấm điểm đo lường (Observability) cho đến báo động tức thời khi sự cố xảy ra (Alerting).
