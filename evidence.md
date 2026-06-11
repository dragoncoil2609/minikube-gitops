# Báo Cáo Thực Hành: GitOps & Ship Smartly (ArgoCD, Rollouts, Canary & SLO)

Tài liệu này tổng hợp chi tiết từng bước thực hành từ việc dựng cụm cơ bản đến thiết lập hệ thống CI/CD & Observability hoàn chỉnh, tuân thủ nghiêm ngặt nguyên tắc GitOps (Mã nguồn là nguồn sự thật duy nhất).

---

## PHẦN 1: NỀN TẢNG GITOPS (BUỔI SÁNG)

### Lab 0: Khởi tạo Cụm và Đẩy Code lên Git
**Mục tiêu:** Cài đặt môi trường Kubernetes cơ bản với Minikube và kết nối với GitHub.
**Các bước thực hiện:**
1. **Khởi tạo cụm K8s local:** Sử dụng Docker driver.
   ```bash
   minikube start -p w9 --driver=docker
   kubectl config use-context w9
   ```
2. **Khởi tạo Git repo:** Tạo thư mục `minikube-gitops`, khởi tạo Git và thiết lập cấu trúc ban đầu.
3. **Đẩy lên GitHub:**
   ```bash
   git init && git add . && git commit -m "init"
   git branch -M main
   git remote add origin https://github.com/dragoncoil2609/minikube-gitops.git
   git push -u origin main
   ```

### Lab 1: Cài đặt ArgoCD vào Cụm
**Mục tiêu:** Đưa "người thợ" ArgoCD vào K8s để theo dõi sát sao Git repo.
**Các bước thực hiện:**
1. Tạo namespace và áp dụng file cài đặt chính thức (dùng cờ `--server-side` để tránh lỗi CRD quá lớn):
   ```bash
   kubectl create ns argocd
   kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
2. Chờ các Pod chạy lên và Port-forward để mở giao diện UI:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
3. Lấy mật khẩu đăng nhập tài khoản `admin`:
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
   ```

### Lab 2 & 3: Khởi tạo Application & Tính năng Self-Heal
**Mục tiêu:** Cấu hình ứng dụng thông qua GitOps và kiểm tra khả năng tự chữa lành.
**Các bước thực hiện:**
1. Khai báo ứng dụng bằng cách tạo file `argocd/apps/reactsurvey.yaml` (trỏ đến thư mục mã nguồn k8s của app).
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: reactsurvey
     namespace: argocd
   spec:
     source:
       repoURL: https://github.com/dragoncoil2609/minikube-gitops.git
       path: apps/reactsurvey
     destination:
       server: https://kubernetes.default.svc
       namespace: reactsurvey
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```
2. **Apply lần đầu:** Dùng lệnh `kubectl apply -f argocd/apps/reactsurvey.yaml` để báo cho ArgoCD biết về ứng dụng này.
3. **Kiểm thử Sync:** Thay đổi số lượng replicas trên Git, Push lên. ArgoCD tự động kéo cấu hình về và cập nhật số Pod.
4. **Kiểm thử Self-Heal:** Thử thao tác tay trên Terminal (`kubectl scale deploy ... --replicas=9`). ArgoCD lập tức nhận diện trạng thái thực tế lệch với Git (OutOfSync) và tự động ép số replicas về đúng như trên file YAML, chặn đứng các can thiệp thủ công.

### Lab 4: Rollback An Toàn Qua Git
**Mục tiêu:** Đưa ứng dụng về phiên bản cũ an toàn khi xảy ra lỗi.
**Các bước thực hiện:**
1. Thay vì dùng `kubectl rollout undo` (cách làm mất đồng bộ với Git), ta sử dụng tính năng Revert của Git:
   ```bash
   git revert HEAD --no-edit
   git push
   ```
2. Sau khi push, mã nguồn trên Git trở về trạng thái của commit trước. ArgoCD đồng bộ sự thay đổi này xuống cụm K8s, hoàn tất quá trình Rollback. Cách này đảm bảo lịch sử minh bạch: "Ai đã rollback, vào lúc nào, tại sao".

### Lab 5: Mô Hình App-of-Apps (Root Application)
**Mục tiêu:** Quản lý tập trung mọi ứng dụng thông qua 1 điểm duy nhất, không bao giờ phải gõ `kubectl apply` nữa.
**Các bước thực hiện:**
1. Tạo file `argocd/root.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata: { name: root, namespace: argocd }
   spec:
     source:
       repoURL: https://github.com/dragoncoil2609/minikube-gitops.git
       path: argocd/apps   # Trỏ vào thư mục chứa các app con
     destination: { server: https://kubernetes.default.svc, namespace: argocd }
     syncPolicy: { automated: { prune: true, selfHeal: true } }
   ```
2. Apply file `root.yaml` thủ công. Từ giờ, thư mục `argocd/apps/` đóng vai trò là danh sách các ứng dụng. Bất cứ khi nào ta thả file yaml định nghĩa ứng dụng vào thư mục này, Root App sẽ tự động phát hiện và đẻ ra ứng dụng đó trên cụm.

### Lab 6: Sync Waves (Thứ Tự Triển Khai)
**Mục tiêu:** Định nghĩa thứ tự ưu tiên khi cài đặt các tài nguyên K8s để tránh lỗi do thiếu phụ thuộc (ví dụ Deployment chạy trước khi có Secret).
**Các bước thực hiện:**
1. Gắn annotation `argocd.argoproj.io/sync-wave` vào các file YAML:
   - `namespace.yaml`: `sync-wave: "-1"` (Chạy đầu tiên để tạo chuồng).
   - `secret.yaml`: `sync-wave: "0"`.
   - `deployment.yaml`: `sync-wave: "1"` (Chỉ khởi chạy khi Secret đã sẵn sàng).
   - `service.yaml`: `sync-wave: "2"`.

### Lab 7: CI (Continuous Integration) Kiểm Tra Mã Nguồn
**Mục tiêu:** Dùng GitHub Actions chặn các file YAML bị lỗi cú pháp ngay từ cổng Pull Request.
**Các bước thực hiện:**
1. Tạo file `.github/workflows/validate.yml` gọi công cụ `kubeconform`.
2. Truy cập GitHub Settings, thiết lập Branch Protection cho nhánh `main`. Yêu cầu mọi PR phải vượt qua bài test `validate` của GitHub Actions mới được phép Merge, tránh cấu hình rác lọt vào cụm.

---

## PHẦN 2: OBSERVABILITY & CANARY (BUỔI CHIỀU)

### Lab 8: Cài đặt Argo Rollouts & Prometheus qua GitOps
**Mục tiêu:** Triển khai các công cụ đo lường và phát hành nâng cao.
**Các bước thực hiện:**
1. Thêm file định nghĩa `argo-rollouts.yaml` và `app-monitoring.yaml` (trỏ đến Helm Chart của Prometheus Stack) vào thư mục `argocd/apps/`.
2. Push lên Git. Root Application sẽ lập tức nhận diện và tự động setup Controller Argo Rollouts cùng hệ thống Kube-Prometheus-Stack vào cụm.

### Lab 9: Chuyển đổi sang Rollout & Cấu hình Canary Thủ Công
**Mục tiêu:** Thay thế luồng RollingUpdate mặc định bằng chiến thuật nhả từ từ lượng truy cập (Canary).
**Các bước thực hiện:**
1. Mở file ứng dụng (ví dụ `api.yaml`), sửa dòng `kind: Deployment` thành `kind: Rollout`.
2. Khai báo chiến lược Canary tạm dừng (Pause) vô thời hạn để kỹ sư test thử:
   ```yaml
   strategy:
     canary:
       steps:
         - setWeight: 25
         - pause: {} # Dừng ở mức 25% traffic và đợi lệnh
   ```
3. **Thao tác tay:** Khi kỹ sư thấy 25% ổn thỏa, họ sẽ gõ lệnh `kubectl argo rollouts promote api -n demo` để nâng lên 100%. Nếu có lỗi, họ gõ `abort` để quay xe.

### Lab 10: Auto-Canary (Tự Động Hóa Hoàn Toàn Bằng Analysis)
**Mục tiêu:** Loại bỏ hoàn toàn sự phụ thuộc vào kỹ sư. Hệ thống tự chấm điểm bản phát hành và tự quyết định Abort nếu có lỗi.
**Các bước thực hiện:**
1. **Định nghĩa bài test:** Tạo `AnalysisTemplate` (`k8s-api/analysis.yaml`) chạy câu query Prometheus đo tỷ lệ HTTP 200.
   ```yaml
   successCondition: result[0] >= 0.90 # Yêu cầu độ ổn định >= 90%
   query: |
     sum(rate(flask_http_request_total{status="200"}[1m])) 
     / sum(rate(flask_http_request_total[1m]))
   ```
2. **Tích hợp vào Canary:** Chèn bước `analysis` vào Rollout.
   ```yaml
         - setWeight: 25
         - analysis:
             templates:
               - templateName: success-rate
   ```
3. **Kết quả:** Khi tung ra một bản cập nhật bị lỗi (`ERROR_RATE=0.5`), bước Analysis sẽ truy vấn Prometheus và phát hiện tỷ lệ thành công bị rớt xuống dưới 90%. Quá trình Canary lập tức bị đánh dấu `Failed`, trạng thái Rollout chuyển sang `Degraded`, hệ thống **tự động Abort** và khôi phục 100% traffic về phiên bản cũ để đảm bảo không sập diện rộng.

### Lab 11: Thiết Lập Cảnh Báo Lỗi Qua Email (Alertmanager)
**Mục tiêu:** Gửi báo động khẩn cấp tới email đội ngũ khi lỗi vượt ngưỡng.
**Các bước thực hiện:**
1. Kích hoạt Alertmanager trong Values của Helm chart Prometheus, cấu hình địa chỉ SMTP của Google (`smtp.gmail.com:587`) và email người nhận.
2. **Bảo mật:** Không lưu mật khẩu trên Git. Khởi tạo Google App Password (mật khẩu ứng dụng gồm 16 chữ cái), sau đó gõ lệnh trực tiếp để chèn mật khẩu này vào một K8s Secret:
   ```bash
   kubectl create secret generic alertmanager-secret --from-literal=smtp-password="[APP_PASSWORD]" -n monitoring
   ```
3. **Viết luật cảnh báo:** Tạo `PrometheusRule` (`alert-rules.yaml`) định nghĩa điều kiện: "Nếu tỷ lệ lỗi HTTP 500 của API vượt mức 5% liên tục trong 1 phút, sẽ kích hoạt nhãn HighErrorRate".
4. **Kết quả:** Khi dùng script `load-generator` bắn hàng ngàn request lỗi 500 vào hệ thống, Prometheus kích hoạt luật cảnh báo, Alertmanager chộp lấy thông tin này và dùng SMTP Secret gửi ngay một email có tựa đề đỏ chót tới hộp thư của nhóm DevOps.
