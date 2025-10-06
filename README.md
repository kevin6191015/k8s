- 使用 ArgoCD 自動進行 k8s 部署
    - kubectl create namespace argocd
    - kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    - kubectl -n argocd get secret argocd-initial-admin-secret   -o jsonpath="{.data.password}" | base64 -d (取得初始帳號 admin 密碼 6drZd5rt6g6PKT9d)
- 部署正式流程

- 新增一個 GHCR 登入 Secret（拉映像用）
- kubectl -n app-prod create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username='你的 GitHub 帳號' \
  --docker-password='具 read:packages 權限的 Personal Access Token' \
  --docker-email='you@example.com'

(1) 程式碼層
- 後端（Spring Boot）
  - 統一環境變數（不要寫死在 yml）
    - SPRING_PROFILES_ACTIVE=prod
    - DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD
    - APP_JWT_SECRET（≥64 bytes，你程式已檢查）
    - APP_JWT_EXP_MINUTES（可選）
    -（同網域反代）SERVER_SERVLET_CONTEXT_PATH=/api
  - 健康檢查端點
    - 開啟 Actuator：management.endpoints.web.exposure.include=health,info
    - 內部 probe 使用 /api/actuator/health（或你自定 /health，二擇一統一）
  - CORS
    - 正式建議不要跨域：用 Ingress 把 /api 反代到後端。這樣後端 CORS 設定可最小化（甚至關掉）。
  - 安全/錯誤
    - 保持 ApiEnvelope 一致；@ControllerAdvice 統一錯誤格式。
    - JwtAuthFilter 跳過 OPTIONS、/auth/**，並加在 UsernamePasswordAuthenticationFilter 前。
    - 密碼一律 Bcrypt；移除明碼 fallback。
  - 資料遷移
    - 使用 Flyway/Liquibase；src/main/resources/db/migration/ 放版控 SQL。
- 前端（Vite）
    - 環境檔
      - .env.production：VITE_API_BASE_URL=/api（同網域反代）
      - 路由
        - createWebHistory(import.meta.env.BASE_URL)；Nginx/Ingress 設 history fallback。
      - HTTP 客戶端
        - Axios 攔截器：自動夾帶 Authorization: Bearer <token>、401 自動導回 /login、統一解包 ApiEnvelope。
      - 安全
        - 移除多餘 console.log；避免 v-html（或加 DOMPurify）。
        - Token 存 localStorage → 嚴格避免 XSS；必要時後端加 CSP。

(2) 容器化（Docker）
- 後端 Dockerfile（多階段）
    - 第一段用 maven build，第二段用 JRE 運行；以 JAVA_OPTS 作 JVM 調整（記憶體等）。
    - 僅 EXPOSE 8080；ENTRYPOINT 用 java -jar app.jar。
- 前端 Dockerfile（build + Nginx serve）
    - Build 階段：npm ci && npm run build
    - Serve 階段：Nginx 映像，拷貝 dist。
    - nginx.conf 設 try_files $uri /index.html;（history fallback）。
    - 注意：容器內不寫入應用目錄，避免 stateful；上傳/匯出等需外部儲存或後端提供下載流。

(3) 建置與映像（CI/CD）
- 建置腳本：
    - 後端：mvn -DskipTests package → docker build -t <registry>/backend:<tag>
    - 前端：npm ci && npm run build → docker build -t <registry>/frontend:<tag>
- 推送：docker push 到私有/公有 registry。
- 標籤策略：使用語義版（1.0.0）＋不可變 digest；禁止覆蓋 latest 在正式環境使用。

(4) Kubernetes 組態（YAML/Helm/Kustomize）
- Namespace
    - 建 app-prod 命名空間。
- Secret / ConfigMap
    - Secret：DB_USER, DB_PASSWORD, APP_JWT_SECRET（不要 commit；可用 Sealed Secrets/External Secrets）
    - ConfigMap：SPRING_PROFILES_ACTIVE, DB_HOST, DB_PORT, DB_NAME, APP_JWT_EXP_MINUTES, SERVER_SERVLET_CONTEXT_PATH
- 注意：JWT 秘鑰務必≥64 bytes；若太短你現有程式會拋例外。

- MySQL（擇一）
    - 推薦：雲端託管（RDS/Cloud SQL），K8s 只連線；減少 Stateful 管理負擔。
    - 自管：StatefulSet + PVC；設定 utf8mb4、持久卷，備份策略另行規劃。

- Backend Deployment + Service
    - replicas: 2 起步，設 resources.requests/limits（例如 250m/1CPU、512Mi/1Gi）。
    - readinessProbe/livenessProbe 指向 /api/actuator/health；初始延遲合理。
    - envFrom 掛 ConfigMap 與 Secret。
    - Service 用 ClusterIP。

- Frontend Deployment + Service
    - replicas: 2；靜態檔服務。
    - Probe 指向 /。
    - Service 用 ClusterIP。

- Ingress（以 Nginx Ingress Controller 為例）
    - 同網域：
        - / → 前端 web-frontend:80
        - /api → 後端 spring-backend:8080
        - 若啟用 TLS：加上 cert-manager（ClusterIssuer）自動簽發/續期。

- 注意：
    -  設 proxy-read-timeout / proxy-send-timeout（長請求時）
    - 若有大檔下載，上調 client_max_body_size（Nginx）。

(5) 資料庫遷移流程
- 選 A（最簡單）：讓 Spring 啟動自動執行 Flyway（有鎖，併發安全）。
- 選 B（更可控）：K8s Job 先跑 migration 成功後，再 rollout 後端。
- 確保多副本下不會重複寫 schema（Flyway 預設會鎖，但請在實際 DB 驗證一次）。

(6) 監控、日誌、擴縮
- HPA：CPU 60% 目標，自動擴縮 2~6 個 Pod（依流量調整）。
- 日誌：輸出 STDOUT（JSON 更佳），集成 EFK/雲端 log。
- 監控：Prometheus/Grafana；後端可加 /actuator/prometheus。
- 追蹤（選配）：OpenTelemetry/Jaeger。

(7) 正式環境安全細節
- CORS：走同網域 /api 反代，避免瀏覽器跨域。
- Headers：前端 Nginx 加安全標頭（CSP/Strict-Transport-Security/X-Frame-Options/Referrer-Policy 等）。
- Token：前端持在 LS → 嚴控 XSS（不使用 v-html；必要時 DOMPurify；後端 CSP）。
- 網路：NetworkPolicy 限制後端只被 Ingress/前端訪問、只出站到 DB。
- 鑰輪替：JWT secret 定期輪替（滾動部署）；DB 密碼也要有換發計畫。
- 檔案上傳：限制大小與副檔名；後端做 MIME 驗證與病毒掃描（若必要）。

(8) 部署與驗證步驟
- 套用 Namespace/Secrets/ConfigMap：kubectl apply -f ns.yaml && -f secrets.yaml && -f config.yaml
-（如需）部署 MySQL 或確認雲端 DB 存取白名單。
- 部署 Backend / Frontend：kubectl apply -f backend.yaml -f frontend.yaml
- 部署 Ingress + TLS：kubectl apply -f ingress.yaml
- 驗證：
    - kubectl -n app-prod get pods,svc,ing
    - kubectl -n app-prod logs deploy/spring-backend（無錯誤）
    - 瀏覽器開 https://你的域名/、打 API https://你的域名/api/health 或 /api/actuator/health
- 壓測與調參：視情況調整 HPA、資源限制、連線池（DataSource pool size）。


- Kustomize
    - 先寫一組通用的 Kubernetes YAML（base），再用 overlay 在不同環境（dev、staging、prod）做少量覆蓋（patch）、替換（image tag）、加註（label/annotation）等，最後由 kubectl apply -k 一次套用。
    - 注意 : Kustomize中的resources只會吃最後一個設定
