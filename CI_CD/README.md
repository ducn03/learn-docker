# 🚀 CI/CD với Docker & GitHub Actions (Java / Spring Boot)

> Tự động hóa: test → build image → push registry → deploy — mỗi khi bạn push code.

---

## Mục lục

1. [CI/CD là gì? Tại sao cần?](#1-cicd-là-gì)
2. [GitHub Actions — Khái niệm cốt lõi](#2-github-actions--khái-niệm-cốt-lõi)
3. [Dockerfile cho Spring Boot](#3-dockerfile-cho-spring-boot)
4. [Workflow cơ bản: Test & Build](#4-workflow-cơ-bản-test--build)
5. [Build & Push Docker Image lên Registry](#5-build--push-docker-image)
6. [Deploy tự động lên Server](#6-deploy-tự-động-lên-server)
7. [Quản lý Secrets & Biến môi trường](#7-quản-lý-secrets--biến-môi-trường)
8. [Workflow hoàn chỉnh Production](#8-workflow-hoàn-chỉnh-production)
9. [Các pattern nâng cao](#9-các-pattern-nâng-cao)
10. [Debug & Troubleshooting](#10-debug--troubleshooting)

---

## 1. CI/CD là gì?

### Không có CI/CD — quy trình thủ công

```
Developer push code
    │
    ▼
Tự chạy: mvn test        ← Dễ quên
    │
    ▼
Tự chạy: mvn package     ← Mất thời gian
    │
    ▼
Tự build: docker build   ← Làm tay, dễ sai
    │
    ▼
Tự push: docker push     ← Cần nhớ tag đúng
    │
    ▼
SSH vào server, pull image, restart  ← Rủi ro cao
```

### Có CI/CD — tự động hoàn toàn

```
Developer push code
    │
    ▼
GitHub Actions tự động:
    ├── ✅ Chạy unit test
    ├── ✅ Build JAR
    ├── ✅ Build Docker image
    ├── ✅ Push lên Docker Hub / GHCR
    └── ✅ Deploy lên server
```

### CI vs CD

| | Ý nghĩa | Làm gì |
|---|---|---|
| **CI** — Continuous Integration | Tích hợp liên tục | Tự động test + build mỗi khi có code mới |
| **CD** — Continuous Delivery | Chuyển giao liên tục | Tự động tạo artifact sẵn sàng deploy |
| **CD** — Continuous Deployment | Triển khai liên tục | Tự động deploy lên production |

---

## 2. GitHub Actions — Khái niệm cốt lõi

### Các thành phần

```
Repository
└── .github/
    └── workflows/
        ├── ci.yml          ← Workflow 1
        └── deploy.yml      ← Workflow 2
```

```yaml
# Cấu trúc một workflow file
name: Tên workflow          # Hiển thị trên GitHub UI

on:                         # TRIGGER — khi nào chạy
  push:
    branches: [main]

jobs:                       # Các công việc (chạy song song mặc định)
  build:                    # Tên job
    runs-on: ubuntu-latest  # Runner — máy ảo để chạy

    steps:                  # Các bước trong job (chạy tuần tự)
      - name: Checkout code
        uses: actions/checkout@v4   # Action có sẵn

      - name: Run my command
        run: echo "Hello!"          # Lệnh shell
```

### Các trigger phổ biến

```yaml
on:
  # Push lên branch cụ thể
  push:
    branches: [main, develop]

  # Tạo Pull Request vào main
  pull_request:
    branches: [main]

  # Chạy theo lịch (cron)
  schedule:
    - cron: '0 2 * * *'   # 2 giờ sáng mỗi ngày

  # Chạy thủ công từ GitHub UI
  workflow_dispatch:

  # Khi tạo tag release
  push:
    tags:
      - 'v*.*.*'
```

### Runners

| Runner | Hệ điều hành | Giá |
|--------|-------------|-----|
| `ubuntu-latest` | Ubuntu 22.04 | Miễn phí (2000 phút/tháng) |
| `windows-latest` | Windows Server | Miễn phí |
| `macos-latest` | macOS | Miễn phí (ít phút hơn) |
| Self-hosted | Máy của bạn | Miễn phí hoàn toàn |

---

## 3. Dockerfile cho Spring Boot

### Dockerfile cơ bản (không tối ưu)

```dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Dockerfile tối ưu với Multi-stage Build

```dockerfile
# ── Stage 1: Build JAR ─────────────────────────────────────
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app

# Copy pom.xml trước — tận dụng Docker layer cache
# Nếu pom.xml không đổi → không cần download dependencies lại
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source và build
COPY src ./src
RUN mvn package -DskipTests -B

# ── Stage 2: Extract layers (Spring Boot 2.3+) ─────────────
FROM builder AS extractor
WORKDIR /app
RUN java -Djarmode=layertools -jar target/*.jar extract

# ── Stage 3: Production image ──────────────────────────────
FROM eclipse-temurin:17-jre-alpine AS production
WORKDIR /app

# Tạo user non-root để bảo mật
RUN addgroup -S spring && adduser -S spring -G spring
USER spring

# Copy các layer theo thứ tự ít thay đổi → hay thay đổi
# (tối ưu cache khi chỉ thay đổi code)
COPY --from=extractor /app/dependencies/ ./
COPY --from=extractor /app/spring-boot-loader/ ./
COPY --from=extractor /app/snapshot-dependencies/ ./
COPY --from=extractor /app/application/ ./

EXPOSE 8080

# Giới hạn memory, bật container-aware GC
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

### Tại sao copy pom.xml trước?

```
Lần 1 build:  pom.xml + source → download deps (3 phút) → compile
Lần 2 build:  chỉ đổi source code
              └── pom.xml không đổi → Docker dùng cache → SKIP download
              └── Chỉ compile lại source (30 giây)
```

### .dockerignore cho Maven project

```
# .dockerignore
target/
.git/
.github/
*.md
.mvn/
mvnw
mvnw.cmd
docker-compose*.yml
.env*
```

---

## 4. Workflow cơ bản: Test & Build

### .github/workflows/ci.yml

```yaml
name: CI — Test & Build

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest

    # Khởi động PostgreSQL để test tích hợp
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      # Bước 1: Lấy code về
      - name: Checkout code
        uses: actions/checkout@v4

      # Bước 2: Cài Java
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      # Bước 3: Cache Maven dependencies
      # Tránh download lại mỗi lần chạy CI
      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      # Bước 4: Chạy tests
      - name: Run tests
        run: mvn test -B
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass

      # Bước 5: Upload test report (xem kết quả trên GitHub UI)
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()   # Upload dù pass hay fail
        with:
          name: test-results
          path: target/surefire-reports/

  build:
    name: Build JAR
    runs-on: ubuntu-latest
    needs: test   # Chỉ chạy nếu job "test" pass

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

      - name: Build JAR (skip tests — đã chạy ở job trước)
        run: mvn package -DskipTests -B

      # Lưu JAR để các job sau dùng
      - name: Upload JAR artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar
          retention-days: 1
```

### Xem kết quả trên GitHub

Sau khi push code, vào **repository → Actions tab** để xem:

```
✅ CI — Test & Build
   ├── ✅ Run Tests        (2m 15s)
   └── ✅ Build JAR        (1m 30s)
```

---

## 5. Build & Push Docker Image

### Chọn Registry

| Registry | Địa chỉ | Phù hợp |
|----------|---------|---------|
| **Docker Hub** | docker.io | Public image, dễ dùng |
| **GHCR** (GitHub Container Registry) | ghcr.io | Tích hợp sẵn với GitHub, private miễn phí |
| **AWS ECR** | *.amazonaws.com | Dùng AWS |

### Workflow build & push lên GHCR (khuyên dùng)

```yaml
name: Build & Push Docker Image

on:
  push:
    branches: [main]
  release:
    types: [published]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}  # vd: username/learn-docker

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write   # Cần để push lên GHCR

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Login vào GHCR — dùng GITHUB_TOKEN tự động, không cần tạo secret
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Tạo metadata: tags và labels cho image
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            # Tag theo branch: main → latest
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
            # Tag theo commit SHA ngắn: sha-abc1234
            type=sha,prefix=sha-,format=short
            # Tag theo semver nếu là release: v1.2.3 → 1.2.3, 1.2, 1
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      # Setup Buildx — hỗ trợ multi-platform và cache
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Build và push image
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          # Cache layer từ registry — tăng tốc build lần sau
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
```

### Kết quả: Image được tạo với các tags

```
ghcr.io/username/learn-docker:latest
ghcr.io/username/learn-docker:sha-abc1234
ghcr.io/username/learn-docker:1.2.3      ← Khi tạo release
```

### Push lên Docker Hub (thay thế)

```yaml
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}   # Dùng Access Token, không phải password!
```

> ⚠️ Docker Hub: Tạo **Access Token** tại Account Settings → Security, không dùng password trực tiếp.

---

## 6. Deploy tự động lên Server

### Phương pháp 1: SSH vào server và pull image mới

Phù hợp cho VPS đơn giản (DigitalOcean, Vultr, EC2...).

```yaml
  deploy:
    name: Deploy to Server
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'   # Chỉ deploy từ branch main

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}        # IP hoặc domain server
          username: ${{ secrets.SERVER_USER }}    # vd: ubuntu
          key: ${{ secrets.SERVER_SSH_KEY }}      # Private key SSH
          script: |
            # Pull image mới nhất
            docker pull ghcr.io/${{ github.repository }}:latest

            # Dừng và xóa container cũ
            docker stop spring-app || true
            docker rm spring-app || true

            # Chạy container mới
            docker run -d \
              --name spring-app \
              --restart unless-stopped \
              -p 8080:8080 \
              -e SPRING_PROFILES_ACTIVE=production \
              -e DATABASE_URL=${{ secrets.DATABASE_URL }} \
              ghcr.io/${{ github.repository }}:latest

            # Kiểm tra container đang chạy
            docker ps | grep spring-app
```

### Thiết lập SSH Key cho GitHub Actions

```bash
# Trên máy local — tạo SSH key pair
ssh-keygen -t ed25519 -C "github-actions" -f ~/.ssh/github_actions

# Copy public key lên server
ssh-copy-id -i ~/.ssh/github_actions.pub user@your-server

# Hoặc thủ công: thêm nội dung public key vào server
cat ~/.ssh/github_actions.pub
# → Thêm dòng này vào ~/.ssh/authorized_keys trên server
```

Sau đó thêm vào GitHub Secrets:
- `SERVER_HOST`: IP hoặc domain server
- `SERVER_USER`: tên user SSH (vd: `ubuntu`)
- `SERVER_SSH_KEY`: Nội dung file `~/.ssh/github_actions` (private key)

### Phương pháp 2: Deploy với Docker Compose trên server

```yaml
      script: |
        cd /opt/myapp

        # Cập nhật image tag trong .env
        echo "IMAGE_TAG=sha-${{ github.sha }}" > .env

        # Pull image và restart
        docker compose pull
        docker compose up -d --no-build

        # Xóa image cũ không dùng
        docker image prune -f
```

---

## 7. Quản lý Secrets & Biến môi trường

### Thêm Secret vào GitHub

**Repository → Settings → Secrets and variables → Actions → New repository secret**

```
DOCKERHUB_USERNAME    = your-dockerhub-username
DOCKERHUB_TOKEN       = dckr_pat_xxxxxxxxxxxx
SERVER_HOST           = 192.168.1.100
SERVER_USER           = ubuntu
SERVER_SSH_KEY        = -----BEGIN OPENSSH PRIVATE KEY-----...
DATABASE_URL          = postgresql://user:pass@host:5432/db
```

### Dùng Secret trong workflow

```yaml
steps:
  - name: Deploy
    env:
      DB_URL: ${{ secrets.DATABASE_URL }}
    run: |
      echo "Deploying with DB: $DB_URL"
      # ⚠️ GitHub Actions tự động mask secrets trong log
      # Output sẽ hiển thị: "Deploying with DB: ***"
```

### Environment — Phân tách staging vs production

```yaml
# Tạo environment tại: Settings → Environments
jobs:
  deploy-staging:
    environment: staging       # Dùng secrets của environment "staging"
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy to ${{ vars.SERVER_URL }}"

  deploy-production:
    environment: production    # Có thể yêu cầu approval thủ công
    needs: deploy-staging
    runs-on: ubuntu-latest
```

### Biến thông thường (không nhạy cảm)

```yaml
# Dùng vars (không bị mask trong log)
env:
  APP_NAME: ${{ vars.APP_NAME }}
  JAVA_VERSION: '17'

# Hoặc khai báo ở cấp workflow
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: myapp
```

---

## 8. Workflow hoàn chỉnh Production

Đây là workflow tổng hợp cho một Spring Boot app thực tế:

### .github/workflows/main.yml

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  release:
    types: [published]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  JAVA_VERSION: '17'

jobs:
  # ── Job 1: Test ───────────────────────────────────────────
  test:
    name: 🧪 Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'

      - name: Cache Maven
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}

      - name: Run tests
        run: mvn test -B
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass
          SPRING_JPA_HIBERNATE_DDL_AUTO: create-drop

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/

  # ── Job 2: Build & Push Image ─────────────────────────────
  build:
    name: 🐳 Build & Push Image
    runs-on: ubuntu-latest
    needs: test
    # Chỉ build image khi push vào main/develop (không phải PR)
    if: github.event_name != 'pull_request'
    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
            type=raw,value=develop,enable=${{ github.ref == 'refs/heads/develop' }}
            type=sha,prefix=sha-,format=short
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

  # ── Job 3: Deploy Staging ─────────────────────────────────
  deploy-staging:
    name: 🚀 Deploy Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment: staging

    steps:
      - name: Deploy to staging server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:develop
            docker stop spring-app-staging || true
            docker rm spring-app-staging || true
            docker run -d \
              --name spring-app-staging \
              --restart unless-stopped \
              -p 8081:8080 \
              -e SPRING_PROFILES_ACTIVE=staging \
              ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:develop
            echo "✅ Deployed to staging"

  # ── Job 4: Deploy Production ──────────────────────────────
  deploy-production:
    name: 🏭 Deploy Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production   # Có thể cấu hình cần approval

    steps:
      - name: Deploy to production server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            docker stop spring-app || true
            docker rm spring-app || true
            docker run -d \
              --name spring-app \
              --restart unless-stopped \
              -p 8080:8080 \
              -e SPRING_PROFILES_ACTIVE=production \
              -e SPRING_DATASOURCE_URL=${{ secrets.DATABASE_URL }} \
              ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

            # Health check sau deploy
            sleep 15
            curl -f http://localhost:8080/actuator/health || exit 1
            echo "✅ Production deployment successful"
```

### Luồng hoàn chỉnh

```
push to develop
    ├── 🧪 Test (postgres service)
    ├── 🐳 Build → ghcr.io/user/app:develop
    └── 🚀 Deploy → staging server (port 8081)

push to main / merge PR
    ├── 🧪 Test
    ├── 🐳 Build → ghcr.io/user/app:latest
    └── 🏭 Deploy → production server (port 8080)

create release v1.2.3
    ├── 🧪 Test
    ├── 🐳 Build → ghcr.io/user/app:1.2.3
    └── 🏭 Deploy → production
```

---

## 9. Các pattern nâng cao

### Health Check sau deploy

Thêm Spring Boot Actuator:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      show-details: never
```

```yaml
# Trong workflow — kiểm tra app đã healthy chưa
- name: Health check
  run: |
    for i in {1..10}; do
      STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://${{ secrets.SERVER_HOST }}:8080/actuator/health)
      if [ "$STATUS" = "200" ]; then
        echo "✅ App is healthy"
        exit 0
      fi
      echo "Attempt $i: status $STATUS, retrying..."
      sleep 10
    done
    echo "❌ Health check failed"
    exit 1
```

### Matrix Build — Test trên nhiều Java version

```yaml
jobs:
  test:
    strategy:
      matrix:
        java: ['17', '21']
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: 'temurin'
      - run: mvn test -B
```

### Reusable Workflow — Tái sử dụng

```yaml
# .github/workflows/reusable-deploy.yml
on:
  workflow_call:          # Có thể được gọi từ workflow khác
    inputs:
      environment:
        required: true
        type: string
    secrets:
      SERVER_HOST:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ${{ inputs.environment }}
        run: echo "Deploying to ${{ inputs.environment }}"
```

```yaml
# Gọi từ workflow khác
jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
    secrets:
      SERVER_HOST: ${{ secrets.PROD_HOST }}
```

### Rollback khi deploy thất bại

```yaml
      script: |
        # Lưu lại image tag đang chạy trước khi update
        CURRENT_TAG=$(docker inspect spring-app --format='{{.Config.Image}}' 2>/dev/null || echo "none")

        docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

        docker stop spring-app || true
        docker rm spring-app || true

        docker run -d --name spring-app -p 8080:8080 \
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

        sleep 15

        # Nếu health check thất bại → rollback
        if ! curl -f http://localhost:8080/actuator/health; then
          echo "❌ Deploy failed, rolling back to $CURRENT_TAG"
          docker stop spring-app || true
          docker rm spring-app || true
          docker run -d --name spring-app -p 8080:8080 "$CURRENT_TAG"
          exit 1
        fi
```

---

## 10. Debug & Troubleshooting

### Xem log chi tiết

```yaml
steps:
  - name: Debug thông tin
    run: |
      echo "Branch: ${{ github.ref }}"
      echo "Commit: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      docker --version
      java --version
```

### Lỗi thường gặp

#### ❌ Permission denied khi push lên GHCR

```
Error: denied: permission_denied: write_package
```

**Giải pháp**: Thêm `permissions` vào job:
```yaml
jobs:
  build:
    permissions:
      packages: write
```

#### ❌ Cache Maven không hoạt động

```yaml
# Sai — key không bao gồm hash của pom.xml
key: ${{ runner.os }}-maven

# Đúng
key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
restore-keys: |
  ${{ runner.os }}-maven-
```

#### ❌ Service postgres chưa sẵn sàng

```yaml
# Thêm options health check
services:
  postgres:
    image: postgres:15
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

#### ❌ SSH connection refused

```bash
# Kiểm tra trên server:
# 1. SSH key đã được thêm vào authorized_keys chưa?
cat ~/.ssh/authorized_keys

# 2. Port SSH có bị firewall chặn không?
sudo ufw status

# 3. Thử kết nối thủ công từ máy local
ssh -i ~/.ssh/github_actions user@server
```

#### ❌ Docker build chậm, không dùng cache

```yaml
# Phải có cả cache-from VÀ cache-to
- uses: docker/build-push-action@v5
  with:
    cache-from: type=registry,ref=ghcr.io/user/app:buildcache
    cache-to: type=registry,ref=ghcr.io/user/app:buildcache,mode=max
```

### Chạy workflow thủ công để test

```yaml
on:
  workflow_dispatch:      # Thêm trigger này
    inputs:
      environment:
        description: 'Deploy to which environment?'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
```

Sau đó vào **Actions → chọn workflow → Run workflow**.

---

## Cấu trúc thư mục hoàn chỉnh

```
learn-docker/
├── .github/
│   └── workflows/
│       ├── ci.yml              ← Test khi có PR
│       └── main.yml            ← Full CI/CD pipeline
├── src/
│   └── main/java/...
├── Dockerfile
├── .dockerignore
├── docker-compose.yml          ← Dùng khi develop local
├── docker-compose.prod.yml     ← Dùng trên server
└── pom.xml
```

---

## Tóm tắt nhanh

```
Code push → GitHub Actions trigger
    ↓
🧪 mvn test (có PostgreSQL service)
    ↓
🐳 docker build (multi-stage, có cache)
    ↓
📦 docker push → ghcr.io
    ↓
🚀 SSH → server → docker pull → docker run
    ↓
✅ Health check /actuator/health
```

---

*Tài liệu thuộc series `learn-docker` — Phần 2: CI/CD với GitHub Actions*