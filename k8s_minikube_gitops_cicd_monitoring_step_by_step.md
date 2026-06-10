# STEP BY STEP: Minikube + GitOps + CI/CD GitHub Actions + Docker Hub + Argo CD + Prometheus/Grafana

> File này viết riêng cho task : **dùng minikube**, có **app repo**, có **gitops repo**, pipeline **build image → push Docker Hub → update tag image trong GitOps repo**, và cài **Prometheus/Grafana thông qua GitOps repo**.

---

## 0. Mục tiêu cuối cùng

Flow cần đạt:

```text
Developer push code vào app repo
        ↓
GitHub Actions chạy CI/CD
        ↓
Build Docker image
        ↓
Push image lên Docker Hub
        ↓
Tự sửa tag image trong GitOps repo
        ↓
Argo CD phát hiện GitOps repo thay đổi
        ↓
Argo CD sync vào Kubernetes local bằng minikube
        ↓
Application chạy trên minikube
        ↓
Prometheus + Grafana được cài bằng GitOps repo
```

Bạn sẽ có 2 repo:

```text
1. app-repo
   - Chứa source code application
   - Chứa Dockerfile
   - Chứa GitHub Actions workflow

2. gitops-repo
   - Chứa YAML Kubernetes của application
   - Chứa Argo CD Application manifest
   - Chứa cấu hình cài Prometheus/Grafana
```

Ví dụ đặt tên repo:

```text
app repo:     minikube-demo-app
gitops repo: minikube-gitops-repo
```

---

# PHẦN 1. Chuẩn bị môi trường local

## 1.1. Công cụ cần có

Cài sẵn:

```text
- Docker Desktop
- kubectl
- minikube
- Git
- Node.js nếu dùng app demo Node.js
- GitHub account
- Docker Hub account
```

Kiểm tra:

```bash
docker version
kubectl version --client
minikube version
git --version
node -v
npm -v
```

Checklist:

```text
[ ] Docker chạy được
[ ] kubectl có version
[ ] minikube có version
[ ] Git có version
[ ] Có GitHub account
[ ] Có Docker Hub account
```

---

# PHẦN 2. Dựng Kubernetes local bằng minikube

## 2.1. Start minikube

```bash
minikube start --driver=docker
```

Nếu máy bạn đã cấu hình driver mặc định, có thể dùng:

```bash
minikube start
```

## 2.2. Kiểm tra minikube

```bash
minikube status
kubectl get nodes
kubectl get pods -A
```

Kết quả mong muốn:

```text
minikube: Running
Node STATUS: Ready
Các pod kube-system Running
```

## 2.3. Test namespace

```bash
kubectl create namespace demo
kubectl get namespace
kubectl delete namespace demo
```

Checklist:

```text
[ ] minikube start thành công
[ ] kubectl get nodes thấy node Ready
[ ] kubectl get pods -A chạy được
[ ] Tạo/xóa namespace demo được
```

---

# PHẦN 3. Tạo app repo

## 3.1. Tạo repo trên GitHub

Tạo repository:

```text
minikube-demo-app
```

Repo này chứa source code app + Dockerfile + GitHub Actions.

## 3.2. Clone app repo

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/minikube-demo-app.git
cd minikube-demo-app
```

Thay `YOUR_GITHUB_USERNAME` bằng GitHub username của bạn.

---

# PHẦN 4. Tạo application mẫu

Nếu bạn đã có app thật thì dùng app thật. Nếu chưa có, dùng app Node.js đơn giản bên dưới.

## 4.1. Tạo package.json

Tạo file `package.json`:

```json
{
  "name": "minikube-demo-app",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

## 4.2. Tạo server.js

Tạo file `server.js`:

```js
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.send('Hello from Minikube GitOps CI/CD demo app - v1');
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

app.listen(port, () => {
  console.log(`Demo app listening on port ${port}`);
});
```

## 4.3. Test app local

```bash
npm install
npm start
```

Mở trình duyệt:

```text
http://localhost:3000
```

Kết quả mong muốn:

```text
Hello from Minikube GitOps CI/CD demo app - v1
```

---

# PHẦN 5. Docker hóa app

## 5.1. Tạo Dockerfile

Tạo file `Dockerfile`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --only=production

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

## 5.2. Tạo .dockerignore

Tạo file `.dockerignore`:

```text
node_modules
.git
.github
README.md
```

## 5.3. Build image local

Thay `YOUR_DOCKERHUB_USERNAME` bằng username Docker Hub của bạn.

```bash
docker build -t YOUR_DOCKERHUB_USERNAME/minikube-demo-app:local .
```

## 5.4. Run container local

```bash
docker run --rm -p 3000:3000 YOUR_DOCKERHUB_USERNAME/minikube-demo-app:local
```

Mở:

```text
http://localhost:3000
```

Checklist:

```text
[ ] Dockerfile tạo xong
[ ] docker build thành công
[ ] docker run chạy được
[ ] App trả response trên port 3000
```

---

# PHẦN 6. Push app repo lên GitHub

```bash
git add .
git commit -m "Initial demo app with Dockerfile"
git branch -M main
git push -u origin main
```

---

# PHẦN 7. Tạo Docker Hub repository

Vào Docker Hub, tạo repository:

```text
minikube-demo-app
```

Image sau này sẽ có dạng:

```text
YOUR_DOCKERHUB_USERNAME/minikube-demo-app:TAG
```

Để làm lab dễ hơn, nên để Docker Hub repo là **public**.

---

# PHẦN 8. Tạo GitOps repo

## 8.1. Tạo repo trên GitHub

Tạo repository mới:

```text
minikube-gitops-repo
```

Repo này sẽ chứa YAML Kubernetes + Argo CD Application cho dự án `reactsurvey` của bạn.

## 8.2. Clone GitOps repo

```bash
cd ..
git clone https://github.com/dragoncoil2609/minikube-gitops.git
cd minikube-gitops
```

## 8.3. Tạo cấu trúc thư mục

Tạo các thư mục chứa cấu hình:

```bash
mkdir -p apps/reactsurvey
mkdir -p argocd
mkdir -p monitoring
```

Cấu trúc mong muốn:

```text
minikube-gitops-repo/
├── apps/
│   └── reactsurvey/
│       ├── namespace.yaml
│       ├── db-deployment.yaml
│       ├── db-service.yaml
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml
│       ├── frontend-deployment.yaml
│       └── frontend-service.yaml
├── argocd/
│   └── app-reactsurvey.yaml
└── monitoring/
    └── app-monitoring.yaml
```

---

# PHẦN 9. Viết Kubernetes YAML cho application

Chúng ta sẽ tạo YAML cho cả Frontend và Backend.

## 9.1. Tạo namespace.yaml

File `apps/reactsurvey/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: reactsurvey
```

## 9.2. Tạo Deployment và Service cho Database (MySQL)

File `apps/reactsurvey/db-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  namespace: reactsurvey
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456"
            - name: MYSQL_DATABASE
              value: "crud_db"
          ports:
            - containerPort: 3306
```

File `apps/reactsurvey/db-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
  namespace: reactsurvey
spec:
  type: ClusterIP
  selector:
    app: db
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
```

## 9.3. Tạo Deployment và Service cho Backend

File `apps/reactsurvey/backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: reactsurvey
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: huyhoang2609/crud_backend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
          env:
            - name: DB_HOST
              value: "db"
            - name: DB_USER
              value: "root"
            - name: DB_PASSWORD
              value: "123456"
            - name: DB_NAME
              value: "crud_db"
            - name: PORT
              value: "3000"
```

File `apps/reactsurvey/backend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: reactsurvey
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 3000
      targetPort: 3000
```

## 9.4. Tạo Deployment và Service cho Frontend

File `apps/reactsurvey/frontend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: reactsurvey
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: huyhoang2609/crud_frontend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80  # Port mặc định của Nginx trong frontend dockerfile
```

File `apps/reactsurvey/frontend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: reactsurvey
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
```

---

# PHẦN 10. Test apply YAML bằng tay trước

Trước khi dùng Argo CD, test YAML có đúng không:

```bash
kubectl apply -f apps/reactsurvey/
kubectl get pods -n reactsurvey
kubectl get svc -n reactsurvey
```

Nếu pod bị `ImagePullBackOff` vì image `latest` chưa tồn tại thì chưa sao. Sau bước GitHub Actions, image thật sẽ được push lên Docker Hub.

Test mở frontend:

```bash
minikube service frontend-service -n reactsurvey
```

Sau khi test xong, xóa để lát nữa Argo CD quản lý lại:

```bash
kubectl delete -f apps/reactsurvey/
```

Checklist:

```text
[ ] kubectl apply không sai cú pháp YAML
[ ] Deployment frontend & backend tạo được
[ ] Service tạo được
```

---

# PHẦN 11. Push GitOps repo lên GitHub

```bash
git add .
git commit -m "Add reactsurvey Kubernetes manifests"
git branch -M main
git push -u origin main
```

---

# PHẦN 12. Cài Argo CD vào minikube

## 12.1. Tạo namespace argocd

```bash
kubectl create namespace argocd
```

## 12.2. Cài Argo CD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 12.3. Kiểm tra pod Argo CD

```bash
kubectl get pods -n argocd
```

Chờ tất cả pod Ready:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

Checklist:

```text
[ ] Namespace argocd tạo thành công
[ ] Argo CD pods Running
[ ] argocd-server Running
```

---

# PHẦN 13. Truy cập Argo CD UI

## 13.1. Port forward Argo CD server

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Mở trình duyệt:

```text
https://localhost:8080
```

## 13.2. Lấy password admin

Git Bash/Linux/macOS:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

PowerShell:

```powershell
$pwd = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($pwd))
```

Login:

```text
Username: admin
Password: password vừa lấy
```

Checklist:

```text
[ ] Port-forward Argo CD thành công
[ ] Mở được https://localhost:8080
[ ] Login được bằng admin
```

---

# PHẦN 14. Tạo Argo CD Application cho reactsurvey

Trong GitOps repo, tạo file `argocd/app-reactsurvey.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: reactsurvey
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/dragoncoil2609/minikube-gitops-repo.git
    targetRevision: main
    path: apps/reactsurvey

  destination:
    server: https://kubernetes.default.svc
    namespace: reactsurvey

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Commit file:

```bash
git add argocd/app-reactsurvey.yaml
git commit -m "Add Argo CD application for reactsurvey"
git push origin main
```

Apply Application vào cluster:

```bash
kubectl apply -f argocd/app-reactsurvey.yaml
```

Kiểm tra:

```bash
kubectl get applications -n argocd
kubectl get pods -n reactsurvey
kubectl get svc -n reactsurvey
```

Trong Argo CD UI cần thấy Application `reactsurvey` chuyển sang `Healthy` và `Synced`.

---

# PHẦN 15. Tạo GitHub token để app repo update GitOps repo

Pipeline trong app repo (`reactsurvey`) cần quyền push vào GitOps repo (`minikube-gitops-repo`).

Tạo GitHub Personal Access Token (PAT) có quyền:

```text
Repository permissions:
- Contents: Read and Write
```

Token này sẽ được dùng trong cấu hình GitHub Secrets.

---

# PHẦN 16. Tạo secrets trong app repo

Vào app repo `reactsurvey` trên GitHub:

```text
Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

Tạo 3 secret:

```text
DOCKERHUB_USERNAME = huyhoang2609
DOCKERHUB_TOKEN    = access token Docker Hub (tạo từ Security settings trong Docker Hub)
GITOPS_TOKEN       = GitHub PAT bạn vừa tạo ở Phần 15
```

---

# PHẦN 17. Tạo GitHub Actions workflow trong app repo

Quay lại app repo `reactsurvey`, tạo/sửa file `.github/workflows/deploy.yml`:

```yaml
name: Build Docker Images and Update GitOps Repo

on:
  push:
    branches:
      - main

jobs:
  build-and-update-gitops:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout app repo
        uses: actions/checkout@v4

      - name: Set image tag
        run: echo "IMAGE_TAG=${GITHUB_SHA::7}" >> $GITHUB_ENV

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Backend
        uses: docker/build-push-action@v6
        with:
          context: ./backend
          push: true
          tags: huyhoang2609/crud_backend:${{ env.IMAGE_TAG }},huyhoang2609/crud_backend:latest

      - name: Build and push Frontend
        uses: docker/build-push-action@v6
        with:
          context: ./frontend
          push: true
          tags: huyhoang2609/crud_frontend:${{ env.IMAGE_TAG }},huyhoang2609/crud_frontend:latest

      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: dragoncoil2609/minikube-gitops-repo
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops-repo

      - name: Update image tags in GitOps repo
        run: |
          cd gitops-repo
          
          # Cập nhật backend
          sed -i "s|image: huyhoang2609/crud_backend:.*|image: huyhoang2609/crud_backend:${IMAGE_TAG}|g" apps/reactsurvey/backend-deployment.yaml
          
          # Cập nhật frontend
          sed -i "s|image: huyhoang2609/crud_frontend:.*|image: huyhoang2609/crud_frontend:${IMAGE_TAG}|g" apps/reactsurvey/frontend-deployment.yaml
          
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add apps/reactsurvey/
          git commit -m "Update frontend & backend images to ${IMAGE_TAG}" || echo "No changes to commit"
          git push
```

Nhớ thay trong workflow:

```text
YOUR_DOCKERHUB_USERNAME → Docker Hub username thật
YOUR_GITHUB_USERNAME    → GitHub username thật
```

Commit workflow:

```bash
git add .github/workflows/deploy.yml
git commit -m "Add CI/CD workflow to build image and update GitOps repo"
git push origin main
```

---

# PHẦN 18. Kiểm tra GitHub Actions chạy

Vào app repo trên GitHub:

```text
Actions
→ Build Docker Image and Update GitOps Repo
```

Cần thấy workflow chạy success.

Các step cần success:

```text
[ ] Checkout app repo
[ ] Login Docker Hub
[ ] Build and push Docker image
[ ] Checkout GitOps repo
[ ] Update image tag in GitOps repo
[ ] Push commit vào GitOps repo
```

Nếu lỗi Docker login:

```text
Kiểm tra DOCKERHUB_USERNAME và DOCKERHUB_TOKEN
```

Nếu lỗi push GitOps repo:

```text
Kiểm tra GITOPS_TOKEN có quyền Contents Read and Write chưa
Kiểm tra đúng tên repository chưa
```

---

# PHẦN 19. Kiểm tra Docker Hub có image mới

Vào Docker Hub repo:

```text
YOUR_DOCKERHUB_USERNAME/minikube-demo-app
```

Cần thấy tag mới dạng:

```text
abc1234
```

Tag này là 7 ký tự đầu của commit SHA.

---

# PHẦN 20. Kiểm tra GitOps repo đã đổi image tag

Vào GitOps repo, mở:

```text
apps/demo-app/deployment.yaml
```

Cần thấy image đổi từ:

```yaml
image: YOUR_DOCKERHUB_USERNAME/minikube-demo-app:latest
```

Thành:

```yaml
image: YOUR_DOCKERHUB_USERNAME/minikube-demo-app:abc1234
```

---

# PHẦN 21. Kiểm tra Argo CD tự sync app vào minikube

Kiểm tra app:

```bash
kubectl get pods -n demo
kubectl describe deployment demo-app -n demo
```

Kiểm tra image đang chạy:

```bash
kubectl get deployment demo-app -n demo -o=jsonpath='{.spec.template.spec.containers[0].image}'
```

Mở app bằng minikube:

```bash
minikube service demo-app-service -n demo
```

Hoặc lấy URL:

```bash
minikube service demo-app-service -n demo --url
```

Checklist:

```text
[ ] Argo CD demo-app Synced
[ ] Argo CD demo-app Healthy
[ ] Pod demo-app Running
[ ] Image tag trong cluster là tag mới
[ ] Mở được app bằng minikube service
```

---

# PHẦN 22. Test update version app

Quay lại app repo, sửa `server.js`:

```js
res.send('Hello from Minikube GitOps CI/CD demo app - v2');
```

Commit và push:

```bash
git add server.js
git commit -m "Update app response to v2"
git push origin main
```

Quan sát flow:

```text
1. GitHub Actions chạy lại
2. Docker Hub có image tag mới
3. GitOps repo deployment.yaml đổi tag mới
4. Argo CD sync lại
5. Pod rollout version mới
6. Mở app thấy v2
```

Kiểm tra rollout:

```bash
kubectl rollout status deployment/demo-app -n demo
kubectl get pods -n demo
```

---

# PHẦN 23. Cài Prometheus/Grafana qua GitOps repo

Yêu cầu là cài Grafana/Prometheus lên Kubernetes thông qua GitOps repo.

Ở đây dùng Argo CD Application để cài Helm chart `kube-prometheus-stack`.

## 23.1. Quay lại GitOps repo

```bash
cd ../minikube-gitops
```

## 23.2. Tạo file monitoring/app-monitoring.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring-stack
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 61.3.2
    helm:
      releaseName: monitoring
      values: |
        grafana:
          adminPassword: admin123
          service:
            type: NodePort

        prometheus:
          service:
            type: NodePort

        alertmanager:
          service:
            type: NodePort

  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

Commit lên GitOps repo:

```bash
git add monitoring/app-monitoring.yaml
git commit -m "Add monitoring stack with Prometheus and Grafana [skip ci]"
git push origin main
```

## 23.3. Apply monitoring Application vào cluster

```bash
kubectl apply -f monitoring/app-monitoring.yaml
```

Kiểm tra:

```bash
kubectl get applications -n argocd
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Trong Argo CD UI cần thấy:

```text
monitoring-stack
Status: Synced
Health: Healthy
```

Checklist:

```text
[ ] monitoring-stack xuất hiện trong Argo CD
[ ] Namespace monitoring được tạo
[ ] Prometheus pod Running
[ ] Grafana pod Running
[ ] Service Grafana tồn tại
```

---

# PHẦN 24. Truy cập Grafana

Port-forward Grafana:

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```

Mở:

```text
http://localhost:3000
```

Login:

```text
Username: admin
Password: admin123
```

Sau khi login, vào:

```text
Dashboards
```

Bạn sẽ thấy dashboard Kubernetes có sẵn.

---

# PHẦN 25. Truy cập Prometheus

Tìm service Prometheus:

```bash
kubectl get svc -n monitoring | grep prometheus
```

Thường service có tên tương tự:

```text
monitoring-kube-prometheus-prometheus
```

Port-forward:

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```

Mở:

```text
http://localhost:9090
```

Test query:

```text
up
```

---

# PHẦN 26. Evidence cần chụp cuối ngày

```text
[Ảnh 1] minikube status
[Ảnh 2] kubectl get nodes
[Ảnh 3] kubectl get pods -A
[Ảnh 4] GitHub Actions workflow success
[Ảnh 5] Docker Hub có image tag mới
[Ảnh 6] GitOps repo deployment.yaml đã đổi image tag
[Ảnh 7] Argo CD demo-app Synced Healthy
[Ảnh 8] kubectl get pods -n demo
[Ảnh 9] App mở được bằng minikube service
[Ảnh 10] Argo CD monitoring-stack Synced Healthy
[Ảnh 11] kubectl get pods -n monitoring
[Ảnh 12] Grafana UI login được
[Ảnh 13] Prometheus query up chạy được
```

---

# PHẦN 27. Lệnh kiểm tra nhanh toàn hệ thống

```bash
minikube status
kubectl get nodes
kubectl get applications -n argocd
kubectl get pods -n demo
kubectl get svc -n demo
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Kiểm tra image trong deployment:

```bash
kubectl get deployment demo-app -n demo -o=jsonpath='{.spec.template.spec.containers[0].image}'
```

Mở app:

```bash
minikube service demo-app-service -n demo
```

Mở Grafana:

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```

Mở Prometheus:

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```

---

# PHẦN 28. Lỗi thường gặp và cách xử lý

## 28.1. Pod bị ImagePullBackOff

Kiểm tra:

```bash
kubectl describe pod -n demo <pod-name>
```

Nguyên nhân thường gặp:

```text
- Sai Docker Hub username
- Sai tên image
- Image tag chưa tồn tại
- Docker Hub repo private nhưng chưa có imagePullSecret
```

Cách xử lý:

```text
- Vào Docker Hub kiểm tra image tag có thật không
- Kiểm tra deployment.yaml đang dùng đúng image không
- Nếu repo private, đổi sang public cho dễ làm lab
```

## 28.2. GitHub Actions lỗi Docker login

Kiểm tra:

```text
- DOCKERHUB_USERNAME đúng chưa
- DOCKERHUB_TOKEN đúng chưa
- Docker Hub token còn hiệu lực không
```

## 28.3. GitHub Actions lỗi push GitOps repo

Kiểm tra:

```text
- GITOPS_TOKEN có quyền Contents Read and Write không
- Token có access đúng repo không
- Tên repo trong workflow đúng chưa
```

## 28.4. Argo CD không sync

Kiểm tra:

```bash
kubectl get applications -n argocd
kubectl describe application demo-app -n argocd
```

Nguyên nhân thường gặp:

```text
- Sai repoURL
- Sai path apps/demo-app
- Repo private nhưng Argo CD chưa có credential
- YAML lỗi cú pháp
```

Nếu repo private, nên đổi GitOps repo sang public khi làm lab cho nhanh.

## 28.5. Grafana pod lâu Running

Chart monitoring khá nặng. Kiểm tra:

```bash
kubectl get pods -n monitoring
kubectl describe pod -n monitoring <pod-name>
```

Nếu minikube thiếu RAM/CPU, restart minikube với nhiều tài nguyên hơn:

```bash
minikube delete
minikube start --driver=docker --cpus=4 --memory=8192
```

Sau đó cài lại Argo CD và apply lại Application.

---

# PHẦN 29. Câu báo cáo ngắn gọn

```text
Hôm nay em đã dựng môi trường Kubernetes local bằng minikube, sau đó triển khai mô hình GitOps với hai repository riêng biệt. App repo chứa source code, Dockerfile và GitHub Actions workflow để tự động build Docker image, push image lên Docker Hub và cập nhật image tag trong GitOps repo. GitOps repo chứa các Kubernetes manifest như Namespace, Deployment và Service. Em cài Argo CD trên minikube để theo dõi GitOps repo và tự động đồng bộ application vào cluster khi có thay đổi. Ngoài ra, em triển khai Prometheus và Grafana thông qua GitOps bằng Argo CD Application sử dụng Helm chart kube-prometheus-stack để phục vụ monitoring cho Kubernetes cluster.
```

---

# PHẦN 30. Checklist hoàn thành task

```text
[ ] Dựng minikube local thành công
[ ] Tạo app repo
[ ] Tạo GitOps repo
[ ] Dockerfile build được image
[ ] GitHub Actions build image thành công
[ ] Image được push lên Docker Hub
[ ] Pipeline sửa được tag image trong GitOps repo
[ ] Argo CD được cài trên minikube
[ ] Argo CD sync app từ GitOps repo
[ ] App chạy được trong namespace demo
[ ] Mở được app bằng minikube service
[ ] Prometheus/Grafana được cài qua GitOps repo
[ ] Grafana truy cập được
[ ] Prometheus truy cập được
[ ] Có evidence để báo cáo
```

---

# PHẦN 31. Tóm tắt flow quan trọng nhất

```text
App repo push code
        ↓
GitHub Actions
        ↓
Docker build
        ↓
Docker push
        ↓
Update deployment.yaml trong GitOps repo
        ↓
Argo CD sync
        ↓
Minikube chạy version mới
```

Đây là phần quan trọng nhất của task.
