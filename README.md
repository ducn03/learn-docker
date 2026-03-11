# 🐳 Học Docker từ A đến Z

> Tài liệu này được viết cho người mới bắt đầu học Docker — không yêu cầu kinh nghiệm trước.

---

## Mục lục

1. [Docker là gì? Tại sao cần dùng?](#1-docker-là-gì)
2. [Cài đặt Docker](#2-cài-đặt-docker)
3. [Các khái niệm cốt lõi](#3-các-khái-niệm-cốt-lõi)
4. [Lệnh Docker cơ bản](#4-lệnh-docker-cơ-bản)
5. [Dockerfile & Image](#5-dockerfile--image)
6. [Networking trong Docker](#6-networking-trong-docker)
7. [Volumes — Lưu trữ dữ liệu](#7-volumes--lưu-trữ-dữ-liệu)
8. [Docker Compose](#8-docker-compose)
9. [Thực hành: Dự án mẫu](#9-thực-hành-dự-án-mẫu)
10. [Lỗi thường gặp & cách xử lý](#10-lỗi-thường-gặp)

---

## 1. Docker là gì?

### Vấn đề trước khi có Docker

Bạn đã từng nghe câu: **"Trên máy tôi chạy được mà!"** chưa?

Đây là vấn đề kinh điển trong lập trình:
- Dev viết code trên macOS → Deploy lên Linux server → Lỗi
- Người A cài Python 3.9, người B cài Python 3.11 → Hành vi khác nhau
- App cần Node.js 18 nhưng server đang chạy Node.js 14

### Docker giải quyết như thế nào?

Docker **đóng gói ứng dụng cùng toàn bộ môi trường** (thư viện, config, runtime) vào một **container** — giống như một chiếc hộp kín. Container này chạy giống hệt nhau ở bất kỳ máy nào có Docker.

```
┌─────────────────────────────────────────┐
│              MÁY TÍNH CỦA BẠN           │
│                                         │
│  ┌──────────────┐  ┌──────────────┐    │
│  │  Container A │  │  Container B │    │
│  │  Node.js 18  │  │  Python 3.11 │    │
│  │  App của bạn │  │  App khác    │    │
│  └──────────────┘  └──────────────┘    │
│                                         │
│           Docker Engine                 │
│                                         │
│         Linux / macOS / Windows         │
└─────────────────────────────────────────┘
```

### Container vs Virtual Machine

| | Container | Virtual Machine |
|---|---|---|
| Kích thước | Vài MB | Vài GB |
| Khởi động | Vài giây | Vài phút |
| Tài nguyên | Nhẹ | Nặng |
| Cô lập | Chia sẻ OS kernel | OS riêng hoàn toàn |
| Phù hợp | Microservices, Dev | Cô lập hoàn toàn |

---

## 2. Cài đặt Docker

### Windows & macOS — Docker Desktop

1. Truy cập: https://www.docker.com/products/docker-desktop
2. Tải bản phù hợp (Windows/macOS Intel/macOS Apple Silicon)
3. Cài đặt theo hướng dẫn
4. Khởi động Docker Desktop

### Ubuntu / Debian

```bash
# Cập nhật package list
sudo apt-get update

# Cài các dependency
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg

# Thêm Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Thêm Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Cài Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Cho phép chạy docker không cần sudo
sudo usermod -aG docker $USER
newgrp docker
```

### Kiểm tra cài đặt

```bash
docker --version
# Docker version 25.0.0, build ...

docker run hello-world
# Nếu thấy "Hello from Docker!" → Cài đặt thành công!
```

---

## 3. Các khái niệm cốt lõi

### 3.1 Image

**Image** là một **bản thiết kế** (template) read-only để tạo container.

- Giống như file `.iso` để cài Windows
- Chứa: OS, runtime, thư viện, code app, config
- Được lưu trên **Docker Hub** (hub.docker.com) hoặc private registry

```bash
# Tải image từ Docker Hub
docker pull nginx

# Xem các image đang có trên máy
docker images
```

### 3.2 Container

**Container** là một **phiên bản đang chạy** của Image.

- Từ 1 image → tạo được nhiều container
- Mỗi container cô lập nhau
- Container có thể start, stop, delete

```
Image (nginx:latest)
    ├── Container 1 (web-server-1, đang chạy, port 8080)
    ├── Container 2 (web-server-2, đang chạy, port 8081)
    └── Container 3 (web-server-3, đã dừng)
```

### 3.3 Docker Hub

Docker Hub là **kho lưu trữ image** trên cloud (như GitHub nhưng cho Docker image).

- **Official images**: `nginx`, `postgres`, `node`, `python` — tin cậy, được maintain bởi nhà phát triển chính thức
- **Community images**: `username/image-name`
- Tìm kiếm tại: https://hub.docker.com

### 3.4 Registry

Registry là nơi lưu trữ và phân phối Docker image.

- **Docker Hub**: Public registry mặc định
- **GitHub Container Registry**: ghcr.io
- **AWS ECR, GCP Artifact Registry**: Registry trên cloud
- **Private registry**: Tự host trong công ty

---

## 4. Lệnh Docker cơ bản

### Quản lý Image

```bash
# Tải image về máy
docker pull ubuntu:22.04

# Xem danh sách image
docker images
docker image ls

# Xóa image
docker rmi ubuntu:22.04
docker image rm ubuntu:22.04

# Tìm kiếm image trên Docker Hub
docker search nginx

# Xem thông tin chi tiết của image
docker inspect nginx
```

### Quản lý Container

```bash
# Chạy container từ image
docker run nginx

# Chạy container ở chế độ nền (detached)
docker run -d nginx

# Chạy với tên tùy chỉnh
docker run -d --name my-nginx nginx

# Chạy và map port: port_máy:port_container
docker run -d -p 8080:80 --name my-nginx nginx

# Xem container đang chạy
docker ps

# Xem tất cả container (kể cả đã dừng)
docker ps -a

# Dừng container
docker stop my-nginx

# Khởi động lại container đã dừng
docker start my-nginx

# Xóa container (phải dừng trước)
docker stop my-nginx && docker rm my-nginx

# Xóa container ngay lập tức
docker rm -f my-nginx

# Xem logs của container
docker logs my-nginx
docker logs -f my-nginx   # Theo dõi log realtime (follow)

# Vào bên trong container (interactive shell)
docker exec -it my-nginx bash
docker exec -it my-nginx sh   # Nếu container không có bash

# Xem thông tin chi tiết container
docker inspect my-nginx

# Xem tài nguyên đang dùng
docker stats
```

### Dọn dẹp

```bash
# Xóa tất cả container đã dừng
docker container prune

# Xóa tất cả image không dùng
docker image prune

# Dọn dẹp toàn bộ (container, image, network, cache)
docker system prune -a
```

---

## 5. Dockerfile & Image

### Dockerfile là gì?

Dockerfile là file văn bản chứa **tập lệnh để tạo ra một Docker image**. Mỗi lệnh trong Dockerfile tạo ra một "layer" trong image.

### Cấu trúc Dockerfile cơ bản

```dockerfile
# ── Bước 1: Chọn image gốc (base image) ──────────────────
FROM node:18-alpine

# ── Bước 2: Thông tin tác giả (tùy chọn) ─────────────────
LABEL maintainer="yourname@email.com"

# ── Bước 3: Đặt thư mục làm việc trong container ─────────
WORKDIR /app

# ── Bước 4: Copy file vào container ───────────────────────
COPY package*.json ./

# ── Bước 5: Chạy lệnh khi build image ─────────────────────
RUN npm install

# ── Bước 6: Copy toàn bộ source code ──────────────────────
COPY . .

# ── Bước 7: Mở port để container lắng nghe ────────────────
EXPOSE 3000

# ── Bước 8: Lệnh chạy khi container khởi động ─────────────
CMD ["node", "server.js"]
```

### Giải thích chi tiết các lệnh

| Lệnh | Mục đích | Ví dụ |
|------|----------|-------|
| `FROM` | Base image (bắt buộc, dòng đầu tiên) | `FROM python:3.11-slim` |
| `WORKDIR` | Đặt thư mục làm việc | `WORKDIR /app` |
| `COPY` | Copy file từ máy host vào image | `COPY . .` |
| `ADD` | Như COPY, thêm hỗ trợ URL và giải nén | `ADD app.tar.gz /app` |
| `RUN` | Chạy lệnh lúc **build** image | `RUN apt-get install -y curl` |
| `CMD` | Lệnh mặc định lúc **chạy** container | `CMD ["npm", "start"]` |
| `ENTRYPOINT` | Lệnh luôn chạy (không override được) | `ENTRYPOINT ["python"]` |
| `ENV` | Đặt biến môi trường | `ENV NODE_ENV=production` |
| `ARG` | Tham số lúc build (không tồn tại khi chạy) | `ARG VERSION=1.0` |
| `EXPOSE` | Khai báo port (chỉ để documentation) | `EXPOSE 8080` |
| `VOLUME` | Khai báo mount point | `VOLUME ["/data"]` |
| `USER` | Đổi user thực thi | `USER node` |

### CMD vs ENTRYPOINT

```dockerfile
# CMD: Có thể bị override khi chạy docker run
CMD ["npm", "start"]
# docker run myapp npm test  → chạy "npm test" thay vì "npm start"

# ENTRYPOINT: Không bị override, CMD trở thành tham số mặc định
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run myapp          → chạy "python app.py"
# docker run myapp test.py  → chạy "python test.py"
```

### .dockerignore

Tương tự `.gitignore`, file này loại trừ các file/folder khi build image:

```
# .dockerignore
node_modules
.git
.env
*.log
dist
coverage
README.md
```

> ⚠️ Luôn tạo `.dockerignore` để tránh copy `node_modules` hoặc thông tin nhạy cảm vào image!

### Build và chạy Image

```bash
# Build image từ Dockerfile trong thư mục hiện tại
docker build -t my-app:1.0 .

# Build với Dockerfile ở vị trí khác
docker build -f ./docker/Dockerfile.prod -t my-app:prod .

# Xem các layer của image
docker history my-app:1.0

# Chạy container từ image vừa build
docker run -d -p 3000:3000 --name my-app-container my-app:1.0
```

### Multi-stage Build — Tối ưu kích thước image

Kỹ thuật này dùng nhiều `FROM`, chỉ giữ lại những gì cần thiết cho production:

```dockerfile
# ── Stage 1: Build ─────────────────────────────────────────
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build   # Tạo ra thư mục /app/dist

# ── Stage 2: Production (chỉ lấy kết quả build) ───────────
FROM node:18-alpine AS production
WORKDIR /app

# Chỉ copy những gì cần thiết từ stage builder
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

RUN npm install --only=production

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Kết quả**: Image production nhỏ hơn nhiều vì không có source code gốc, dev dependencies, hay build tools.

---

## 6. Networking trong Docker

### Các loại network

Docker tạo sẵn 3 network mặc định:

```bash
docker network ls
# NETWORK ID   NAME      DRIVER    SCOPE
# abc123       bridge    bridge    local
# def456       host      host      local
# ghi789       none      null      local
```

#### 1. Bridge Network (mặc định)

- Mặc định khi chạy container
- Các container trong cùng bridge network có thể nói chuyện với nhau qua **tên container**
- Container được cô lập với mạng bên ngoài (cần `-p` để expose port)

```bash
# Chạy container — tự động dùng bridge network
docker run -d --name web nginx

# Tạo bridge network tùy chỉnh (khuyên dùng)
docker network create my-network

# Kết nối container vào network tùy chỉnh
docker run -d --name web --network my-network nginx
docker run -d --name api --network my-network node-app

# Container "api" có thể gọi container "web" bằng tên:
# http://web:80
```

#### 2. Host Network

- Container dùng chung network interface với máy host
- Không cần map port, truy cập trực tiếp
- Chỉ hoạt động trên Linux

```bash
docker run -d --network host nginx
# Nginx lắng nghe trực tiếp port 80 của máy host
```

#### 3. None Network

- Container hoàn toàn cô lập, không có network
- Dùng cho các task cần bảo mật cao

```bash
docker run --network none my-secure-job
```

### Các lệnh quản lý Network

```bash
# Tạo network mới
docker network create my-network

# Xem danh sách network
docker network ls

# Xem chi tiết network (ai đang kết nối)
docker network inspect my-network

# Kết nối container đang chạy vào network
docker network connect my-network my-container

# Ngắt kết nối
docker network disconnect my-network my-container

# Xóa network
docker network rm my-network
```

### Giao tiếp giữa các container

```
┌─────────────────── my-network ──────────────────────┐
│                                                      │
│  ┌─────────────┐     http://api:3000    ┌──────────┐ │
│  │   frontend  │ ──────────────────────▶│   api   │ │
│  │  (nginx)    │                        │ (node)  │ │
│  └─────────────┘                        └────┬────┘ │
│                                              │       │
│                              postgres://db:5432      │
│                                              ▼       │
│                                        ┌──────────┐  │
│                                        │    db    │  │
│                                        │(postgres)│  │
│                                        └──────────┘  │
└──────────────────────────────────────────────────────┘
```

> 💡 Khi các container cùng trong một **user-defined bridge network**, chúng có thể gọi nhau bằng **tên container** — không cần biết IP.

---

## 7. Volumes — Lưu trữ dữ liệu

### Vấn đề với dữ liệu trong container

Container là **ephemeral** (tạm thời) — khi container bị xóa, **mọi dữ liệu bên trong cũng mất**!

```bash
# Ví dụ về việc mất dữ liệu
docker run -d --name test-db postgres
# ... lưu dữ liệu vào database ...
docker rm -f test-db   # Xóa container → Dữ liệu mất!
```

### Giải pháp: 3 loại Storage

#### 1. Named Volume (Khuyên dùng)

Docker quản lý storage, lưu ở `/var/lib/docker/volumes/` trên Linux.

```bash
# Tạo volume
docker volume create my-data

# Dùng volume khi chạy container
docker run -d \
  --name postgres \
  -v my-data:/var/lib/postgresql/data \
  postgres:15

# Xem danh sách volume
docker volume ls

# Xem chi tiết volume
docker volume inspect my-data

# Xóa volume (cẩn thận!)
docker volume rm my-data

# Xóa volume không dùng
docker volume prune
```

#### 2. Bind Mount

Map thư mục trên máy host vào container — rất hữu ích khi **develop** (thay đổi code ngay lập tức):

```bash
# Cú pháp: -v /đường/dẫn/host:/đường/dẫn/container
docker run -d \
  --name my-app \
  -v $(pwd)/src:/app/src \
  -p 3000:3000 \
  my-app

# Thay đổi file trong ./src → container thấy ngay lập tức!
```

#### 3. tmpfs Mount

Lưu dữ liệu tạm trong RAM — tự xóa khi container dừng:

```bash
docker run -d \
  --tmpfs /tmp \
  my-app
```

### So sánh các loại Storage

| | Named Volume | Bind Mount | tmpfs |
|---|---|---|---|
| Vị trí | Docker quản lý | Thư mục host | RAM |
| Persistence | ✅ Còn sau khi xóa container | ✅ Tùy thuộc vào host | ❌ Mất khi container dừng |
| Dùng cho | Production data | Development | Dữ liệu nhạy cảm tạm thời |
| Chia sẻ giữa containers | ✅ Dễ dàng | ✅ Qua đường dẫn | ❌ |

### Backup và Restore Volume

```bash
# Backup volume vào file tar
docker run --rm \
  -v my-data:/data \
  -v $(pwd):/backup \
  ubuntu \
  tar czf /backup/my-data-backup.tar.gz /data

# Restore từ backup
docker run --rm \
  -v my-data:/data \
  -v $(pwd):/backup \
  ubuntu \
  tar xzf /backup/my-data-backup.tar.gz -C /
```

---

## 8. Docker Compose

### Docker Compose là gì?

Docker Compose cho phép bạn **định nghĩa và chạy nhiều container cùng lúc** bằng một file YAML duy nhất.

**Không có Docker Compose**: Cần chạy nhiều lệnh dài dòng
```bash
docker network create app-network
docker volume create db-data
docker run -d --name db --network app-network -v db-data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres:15
docker run -d --name api --network app-network -p 3000:3000 -e DATABASE_URL=postgresql://postgres:secret@db:5432/mydb my-api
docker run -d --name web --network app-network -p 80:80 my-frontend
```

**Với Docker Compose**: Một lệnh duy nhất
```bash
docker compose up -d
```

### File docker-compose.yml

```yaml
# docker-compose.yml
version: "3.9"

services:
  # ── Service 1: Database ────────────────────────────────
  db:
    image: postgres:15-alpine
    container_name: my-postgres
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret123
      POSTGRES_DB: myapp
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Chạy SQL khi khởi tạo
    ports:
      - "5432:5432"    # Expose để dùng GUI như DBeaver
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Service 2: Backend API ─────────────────────────────
  api:
    build:
      context: ./backend   # Thư mục chứa Dockerfile
      dockerfile: Dockerfile
    container_name: my-api
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://admin:secret123@db:5432/myapp
      JWT_SECRET: my-jwt-secret
    ports:
      - "3000:3000"
    volumes:
      - ./backend:/app          # Bind mount cho hot-reload
      - /app/node_modules       # Giữ node_modules của container
    depends_on:
      db:
        condition: service_healthy   # Đợi db healthy mới chạy
    restart: unless-stopped

  # ── Service 3: Frontend ────────────────────────────────
  web:
    build: ./frontend
    container_name: my-frontend
    ports:
      - "80:80"
    depends_on:
      - api
    restart: unless-stopped

  # ── Service 4: Redis Cache ─────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: my-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

# ── Volumes ────────────────────────────────────────────────
volumes:
  db-data:
  redis-data:

# ── Networks ───────────────────────────────────────────────
networks:
  default:
    name: my-app-network
```

### Các lệnh Docker Compose

```bash
# Khởi động toàn bộ services (detached)
docker compose up -d

# Khởi động và rebuild image trước
docker compose up -d --build

# Dừng và xóa containers (giữ volumes)
docker compose down

# Dừng và xóa cả volumes (⚠️ mất dữ liệu!)
docker compose down -v

# Xem logs tất cả services
docker compose logs

# Xem logs của một service cụ thể, theo dõi realtime
docker compose logs -f api

# Xem trạng thái các services
docker compose ps

# Chạy lệnh trong container đang chạy
docker compose exec api bash
docker compose exec db psql -U admin myapp

# Chạy một lần (tạo container mới, tự xóa sau khi xong)
docker compose run --rm api npm run migrate

# Restart một service
docker compose restart api

# Scale service (chạy nhiều instance)
docker compose up -d --scale api=3

# Xem cấu hình đã được resolve
docker compose config
```

### Sử dụng file .env với Docker Compose

```bash
# .env
POSTGRES_USER=admin
POSTGRES_PASSWORD=secret123
API_PORT=3000
```

```yaml
# docker-compose.yml — dùng biến từ .env
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  api:
    ports:
      - "${API_PORT}:3000"
```

### Multiple Compose Files (Dev vs Production)

```bash
# docker-compose.yml (base)
# docker-compose.override.yml (dev — tự động load)
# docker-compose.prod.yml (production)

# Development (tự động merge yml + override.yml)
docker compose up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

```yaml
# docker-compose.override.yml (chỉ dùng khi dev)
services:
  api:
    volumes:
      - ./backend:/app   # Hot-reload
    environment:
      NODE_ENV: development

# docker-compose.prod.yml (production)
services:
  api:
    image: registry.example.com/my-api:latest   # Dùng image đã build
    environment:
      NODE_ENV: production
    restart: always
```

---

## 9. Thực hành: Dự án mẫu

Hãy tạo một ứng dụng Todo List với Node.js + PostgreSQL + Nginx.

### Cấu trúc thư mục

```
learn-docker/
├── docker-compose.yml
├── .env
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── Dockerfile
│   └── index.html
└── nginx/
    └── nginx.conf
```

### backend/server.js

```javascript
const express = require('express');
const { Pool } = require('pg');

const app = express();
app.use(express.json());

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Tạo bảng nếu chưa có
pool.query(`
  CREATE TABLE IF NOT EXISTS todos (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    done BOOLEAN DEFAULT false
  )
`);

app.get('/api/todos', async (req, res) => {
  const result = await pool.query('SELECT * FROM todos ORDER BY id');
  res.json(result.rows);
});

app.post('/api/todos', async (req, res) => {
  const { title } = req.body;
  const result = await pool.query(
    'INSERT INTO todos (title) VALUES ($1) RETURNING *',
    [title]
  );
  res.json(result.rows[0]);
});

app.listen(3000, () => console.log('API running on port 3000'));
```

### backend/Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

### docker-compose.yml

```yaml
version: "3.9"

services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: tododb
      POSTGRES_USER: todo_user
      POSTGRES_PASSWORD: todo_pass
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U todo_user -d tododb"]
      interval: 5s
      retries: 5

  api:
    build: ./backend
    environment:
      DATABASE_URL: postgresql://todo_user:todo_pass@db:5432/tododb
    depends_on:
      db:
        condition: service_healthy

  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./frontend:/usr/share/nginx/html
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - api

volumes:
  db-data:
```

### Chạy dự án

```bash
# Clone repo và vào thư mục
cd learn-docker

# Build và khởi chạy
docker compose up -d --build

# Kiểm tra
docker compose ps
docker compose logs api

# Truy cập http://localhost
```

---

## 10. Lỗi thường gặp

### Lỗi 1: Port đã được sử dụng

```
Error: Bind for 0.0.0.0:3000 failed: port is already allocated
```

**Giải pháp:**
```bash
# Tìm process đang dùng port
lsof -i :3000          # macOS/Linux
netstat -ano | findstr :3000  # Windows

# Đổi port trong docker-compose.yml
ports:
  - "3001:3000"   # Đổi 3000 thành 3001 ở phía host
```

### Lỗi 2: Container exit ngay lập tức

```bash
docker ps -a  # Thấy container ở trạng thái "Exited"
```

**Giải pháp:**
```bash
# Xem logs để tìm nguyên nhân
docker logs ten-container

# Chạy interactively để debug
docker run -it --entrypoint sh my-image
```

### Lỗi 3: Cannot connect to the Docker daemon

```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

**Giải pháp:**
```bash
# Kiểm tra Docker có đang chạy không
sudo systemctl status docker

# Khởi động Docker
sudo systemctl start docker

# macOS/Windows: Mở Docker Desktop
```

### Lỗi 4: No space left on device

```
no space left on device
```

**Giải pháp:**
```bash
# Xem dung lượng Docker đang dùng
docker system df

# Dọn dẹp
docker system prune -a --volumes
```

### Lỗi 5: Container không kết nối được với container khác

**Nguyên nhân**: Hai container khác network.

**Giải pháp:**
```bash
# Kiểm tra network
docker inspect container-a | grep NetworkMode
docker inspect container-b | grep NetworkMode

# Dùng docker-compose → tự động cùng network
# Hoặc tạo network chung và kết nối cả hai
docker network create shared-net
docker network connect shared-net container-a
docker network connect shared-net container-b
```

---

## Tham khảo thêm

| Tài nguyên | Link |
|-----------|------|
| Docker Docs chính thức | https://docs.docker.com |
| Docker Hub | https://hub.docker.com |
| Play with Docker (thực hành online) | https://labs.play-with-docker.com |
| Docker Compose Docs | https://docs.docker.com/compose |

---

*Tài liệu được tạo cho repo `learn-docker` — cập nhật lần cuối: 2026*