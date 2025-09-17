# 🚀 Triển khai Docker Private Registry trên AWS EC2

Tài liệu này hướng dẫn triển khai **Docker Private Registry** trên AWS EC2.

---

## 📋 Mục lục
1. [Chuẩn bị hạ tầng](#1-chuẩn-bị-hạ-tầng)
2. [Chuẩn bị Domain + DNS](#2-chuẩn-bị-domain--dns)
3. [Cấu hình quyền AWS (S3 Backend)](#3-cấu-hình-quyền-aws-s3-backend)
4. [Cài đặt phần mềm trên EC2](#4-cài-đặt-phần-mềm-trên-ec2)
5. [Authentication](#5-authentication)
6. [Cấu hình Registry](#6-cấu-hình-registry)
7. [Docker Compose](#7-docker-compose)
8. [Nginx + SSL](#8-nginx--ssl)
9. [Khởi chạy Registry](#9-khởi-chạy-registry)
10. [Client Side (Dev / CI/CD)](#10-client-side-dev--cicd)
11. [Bảo trì & Vận hành](#11-bảo-trì--vận-hành)
12. [Checklist nhanh](#-checklist-nhanh)

---

## 1. Chuẩn bị hạ tầng

### Tạo các tài nguyên AWS:
- **VPC / Subnet / Security Group**
- **EC2 Instance (Ubuntu)**, gán **Elastic IP**
- **Key Pair (.pem)** để SSH vào EC2

### Cấu hình Security Group - mở các port:
- `22` (SSH)
- `80` (HTTP – cấp chứng chỉ SSL)
- `443` (HTTPS – registry)
- `5000` (tuỳ chọn) nếu expose trực tiếp registry

---

## 2. Chuẩn bị Domain + DNS

1. Đăng ký domain/subdomain: `registry.company.com`
2. Tạo bản ghi **A record** → Elastic IP của EC2
3. Kiểm tra DNS phân giải đúng:
   ```bash
   nslookup registry.company.com
   ```

---

## 3. Cấu hình quyền AWS (S3 Backend)

### Tạo S3 Bucket và IAM:
1. Tạo **S3 Bucket** để lưu images
2. Tạo **IAM Policy** với các quyền:
   - `s3:PutObject`
   - `s3:GetObject`
   - `s3:DeleteObject`
   - `s3:ListBucket`
3. Gán policy cho **IAM Role** attach EC2 (khuyến nghị)
4. (Tuỳ chọn) Nếu không dùng role: tạo **IAM User** → lấy **Access Key / Secret Key**

---

## 4. Cài đặt phần mềm trên EC2

SSH vào EC2 và chạy các lệnh sau:

```bash
sudo apt update -y
sudo apt install -y docker.io docker-compose nginx certbot python3-certbot-nginx apache2-utils
sudo systemctl enable --now docker

# Thêm user hiện tại vào group docker
sudo usermod -aG docker $USER
newgrp docker
```

---

## 5. Authentication

Tạo thư mục và file authentication:

```bash
mkdir -p auth
docker run --rm --entrypoint htpasswd registry:2 -Bbn deployer StrongPassword123! > auth/htpasswd
```

> **Lưu ý**: Thay đổi `deployer` và `StrongPassword123!` thành username và password mong muốn.

---

## 6. Cấu hình Registry

Tạo file `config.yml`:

```yaml
version: 0.1
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  # S3 backend (khuyến nghị):
  # s3:
  #   region: ap-southeast-1
  #   bucket: my-registry-bucket
  #   accesskey: <AWS_ACCESS_KEY_ID>
  #   secretkey: <AWS_SECRET_ACCESS_KEY>
  #   secure: true

http:
  addr: :5000

auth:
  htpasswd:
    realm: "Registry Realm"
    path: /auth/htpasswd

delete:
  enabled: true
```

### Để sử dụng S3 backend, uncomment và cấu hình phần S3:
```yaml
storage:
  s3:
    region: ap-southeast-1
    bucket: your-registry-bucket-name
    # Nếu sử dụng IAM Role, bỏ qua accesskey và secretkey
    # accesskey: <AWS_ACCESS_KEY_ID>
    # secretkey: <AWS_SECRET_ACCESS_KEY>
    secure: true
```

---

## 7. Docker Compose

Tạo file `docker-compose.yml`:

```yaml
version: '3.7'
services:
  registry:
    image: registry:2
    restart: always
    container_name: registry-server
    ports:
      - "5000:5000"
    environment:
      REGISTRY_HTTP_ADDR: :5000
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: "Registry Realm"
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    volumes:
      - ./data:/var/lib/registry
      - ./auth:/auth:ro
      - ./config.yml:/etc/docker/registry/config.yml:ro
```

Tạo thư mục data:
```bash
mkdir -p data
```

---

## 8. Nginx + SSL

### Tạo cấu hình Nginx

Tạo file `/etc/nginx/sites-available/registry.conf`:

```nginx
server {
    listen 80;
    server_name registry.company.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name registry.company.com;

    ssl_certificate /etc/letsencrypt/live/registry.company.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/registry.company.com/privkey.pem;

    client_max_body_size 0;

    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Kích hoạt site và cấp chứng chỉ SSL:

```bash
# Kích hoạt site
sudo ln -s /etc/nginx/sites-available/registry.conf /etc/nginx/sites-enabled/
sudo nginx -t

# Cấp chứng chỉ SSL
sudo certbot --nginx -d registry.company.com

# Reload nginx
sudo systemctl reload nginx
```

---

## 9. Khởi chạy Registry

```bash
# Khởi chạy registry
docker-compose up -d

# Kiểm tra nginx config và reload
sudo nginx -t && sudo systemctl reload nginx

# Kiểm tra services
docker-compose ps
sudo systemctl status nginx
```

### Kiểm tra hoạt động:

```bash
# Kiểm tra API catalog
curl -k https://registry.company.com/v2/_catalog

# Kiểm tra với authentication
curl -u deployer:StrongPassword123! https://registry.company.com/v2/_catalog
```

---

## 10. Client Side (Dev / CI/CD)

### Đăng nhập từ máy client:

```bash
docker login registry.company.com
# Nhập username: deployer
# Nhập password: StrongPassword123!
```

### Push image lên registry:

```bash
# Tag image
docker tag myapp:latest registry.company.com/myteam/myapp:1.0.0

# Push image
docker push registry.company.com/myteam/myapp:1.0.0
```

### Pull image từ registry:

```bash
docker pull registry.company.com/myteam/myapp:1.0.0
```

---

## 11. Bảo trì & Vận hành

### 🔄 Backup

**Filesystem backend:**
```bash
# Snapshot EBS volume
aws ec2 create-snapshot --volume-id vol-xxxxxxxxx --description "Registry backup $(date)"
```

**S3 backend:**
- Bật versioning cho S3 bucket
- Thiết lập lifecycle policy để quản lý old versions

### 🗑️ Garbage Collection

```bash
# Dừng registry
docker-compose down

# Chạy garbage collection
docker run --rm -v $(pwd)/data:/var/lib/registry registry:2 garbage-collect /etc/docker/registry/config.yml

# Khởi động lại
docker-compose up -d
```

### 📊 Monitoring

**Giám sát hệ thống:**
- CPU, RAM, Disk usage
- S3 storage usage (nếu dùng S3 backend)
- Network traffic

**Thu thập logs:**
```bash
# Registry logs
docker-compose logs -f registry

# Nginx logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

**Gửi logs lên CloudWatch:**
```bash
# Cài đặt CloudWatch agent
sudo snap install amazon-cloudwatch-agent

# Cấu hình để gửi logs
```

### 📈 Scaling

**Tùy chọn mở rộng:**
1. **Load Balancer + nhiều EC2 instances**
2. **Sử dụng AWS ECR** cho nhu cầu enterprise
3. **Harbor** cho full-featured registry với UI

---

## ✅ Checklist nhanh

- [ ] **EC2 + Elastic IP + Security Group**
- [ ] **Domain + DNS** (A record pointing to Elastic IP)
- [ ] **S3 Bucket + IAM Role** (nếu sử dụng S3 backend)
- [ ] **Cài Docker + Compose + Nginx + Certbot**
- [ ] **Authentication** (htpasswd file)
- [ ] **Registry config** (config.yml + docker-compose.yml)
- [ ] **Nginx reverse proxy + SSL** certificate
- [ ] **Khởi chạy registry** (docker-compose up -d)
- [ ] **Test login + push/pull** từ client
- [ ] **Thiết lập backup + monitoring + GC** schedule

---

## 🔧 Troubleshooting

### Common Issues:

**1. SSL Certificate không hoạt động:**
```bash
sudo certbot renew --dry-run
sudo nginx -t
sudo systemctl reload nginx
```

**2. Registry không thể push large images:**
```bash
# Tăng client_max_body_size trong nginx config
client_max_body_size 0;  # Unlimited
```

**3. Authentication failed:**
```bash
# Tạo lại htpasswd file
docker run --rm --entrypoint htpasswd registry:2 -Bbn username password > auth/htpasswd
docker-compose restart
```

**4. S3 permissions error:**
```bash
# Kiểm tra IAM role/policy
aws sts get-caller-identity
aws s3 ls s3://your-bucket-name/
```

---

## 📚 Tài liệu tham khảo

- [Docker Registry Official Documentation](https://docs.docker.com/registry/)
- [Docker Registry Configuration Reference](https://docs.docker.com/registry/configuration/)
- [Let's Encrypt SSL Certificate](https://letsencrypt.org/getting-started/)
- [AWS S3 Storage Backend](https://docs.docker.com/registry/storage-drivers/s3/)

---

**🎯 Kết luận**: Registry này phù hợp cho team nhỏ đến vừa (10-100 developers). Đối với nhu cầu lớn hơn, cân nhắc sử dụng AWS ECR hoặc Harbor để có thêm các tính năng như vulnerability scanning, image replication, và web UI.
