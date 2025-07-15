# Traefik Reverse Proxy với Docker Compose

> Thay vì sử dụng Docker label cho từng container, dự án này sử dụng cấu hình file cho Traefik. Cách này dễ bảo trì, không cần khởi động lại container khi thay đổi routing hoặc middleware, rất phù hợp cho server chạy nhiều dự án Docker Compose và tăng tính bảo mật.

Hướng dẫn này giúp bạn triển khai Traefik v3 reverse proxy sẵn sàng cho môi trường production bằng Docker Compose. Bao gồm SSL tự động với Let's Encrypt, cấu hình động, và cách thêm site/app mới.

---

**Tại sao nên dùng cấu hình file?**
- Dễ bảo trì hơn Docker label
- Không cần khởi động lại container khi thay đổi routing/middleware
- Phù hợp cho server chạy nhiều Docker Compose
- Bảo mật và tách biệt rõ ràng

---

## Các kịch bản sử dụng

Bạn có thể triển khai Traefik theo hai cách chính, tuỳ nhu cầu:

### 1. Reverse Proxy tiêu chuẩn (cho ứng dụng)
- Dùng `docker-compose.yml` để chạy Traefik reverse proxy cho các ứng dụng của bạn.
- Mở các port 80, 443, 443/udp cho HTTP/HTTPS.
- Phù hợp với hầu hết các trường hợp: app truy cập qua domain hoặc IP server.
- Khởi động với:
  ```bash
  docker-compose up -d
  ```

### 2. Dashboard trên subdomain riêng
- Dùng `docker-compose.dashboard.yml` nếu muốn dashboard Traefik truy cập qua subdomain riêng (ví dụ: traefik.tenmiencuaban.com).
- Mở các port 80, 443, 443/udp (không cần mở 3080:8080).
- Cập nhật subdomain trong `dynamic/sites/traefik-dashboard.yml` và trỏ DNS về server.
- Khởi động với:
  ```bash
  docker-compose -f docker-compose.dashboard.yml up -d
  ```

---

## Cấu trúc thư mục

```
traefik/
├── docker-compose.yml               # File Docker Compose chính cho Traefik (reverse proxy cho app)
├── docker-compose.dashboard.yml     # File Docker Compose cho dashboard Traefik trên subdomain
├── traefik.yml                      # Cấu hình tĩnh cho Traefik
├── letsencrypt/                     # Lưu chứng chỉ SSL (acme.json)
│   └── acme.json
├── dynamic/                         # Thư mục cấu hình động
│   ├── middlewares/                 # Các middleware dùng chung (gzip, rate-limit, ...)
│   │   ├── cache-static.yml
│   │   ├── error-pages.yml
│   │   ├── gzip.yml
│   │   ├── rate-limit.yml
│   │   ├── redirect-https.yml
│   │   ├── secure-headers.yml
│   │   └── whitelist.yml
│   └── sites/                       # Cấu hình cho từng site
│       ├── _template-site.yml
│       └── traefik-dashboard.yml    # Router cho dashboard Traefik trên subdomain
└── README.md                        # Tài liệu dự án
```

---

## Yêu cầu
- Đã cài Docker & Docker Compose
- Có domain trỏ về IP server

---

## 1. Clone repository
```bash
gh repo clone thienphuc2611/traefik-reverse-proxy-with-docker-compose
cd traefik-reverse-proxy-with-docker-compose
```

---

## 2. (Tuỳ chọn) Đổi tên thư mục
Nếu muốn tên thư mục ngắn gọn:
```bash
mv traefik-reverse-proxy-with-docker-compose traefik
cd traefik
```

---

## 3. Cấu hình Traefik
- Mở `docker-compose.yml` và sửa email của bạn cho Let's Encrypt ở phần `command:`:
  ```yaml
  - --certificatesresolvers.letsencrypt.acme.email=your@email.com
  ```
- (Tuỳ chọn) Sửa `traefik.yml` nếu muốn cấu hình tĩnh nâng cao.

---

## 4. Chuẩn bị lưu trữ SSL
```bash
mkdir -p letsencrypt
chmod 600 letsencrypt/acme.json || touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json
```

---

## 5. Tạo network cho Traefik
Nếu dùng external network trong `docker-compose.yml`, cần tạo trước khi start Traefik:
```bash
docker network create traefik-network
```

---

## 6. Khởi động Traefik
```bash
docker-compose up -d
```
- Traefik sẽ lắng nghe trên port 80 (HTTP) và 443 (HTTPS).
- Dashboard có thể truy cập tại https://<your-domain>:8080 (nếu bật).

---

## 7. Thêm site mới
1. **Copy template:**
   ```bash
   cp dynamic/sites/_template-site.yml dynamic/sites/example.com.yml
   ```
2. **Sửa file `example.com.yml`:**
   - Đặt `rule: "Host(`example.com`)"`
   - Đặt đúng tên service và container
   - Ví dụ:
     ```yaml
     http:
       routers:
         example:
           rule: "Host(`example.com`)"
           entryPoints:
             - websecure
           tls:
             certResolver: letsencrypt
           middlewares:
             - redirect-https@file
             - secure-headers@file
             - gzip@file
             - rate-limit@file
             - cache-static@file
           service: example
       services:
         example:
           loadBalancer:
             servers:
               - url: "http://your-app-container:80"
     ```
3. **Đảm bảo container app nằm cùng network với Traefik** (xem bước tiếp theo).

---

## 8. Kết nối app vào network Traefik
Trong `docker-compose.yml` của app, thêm:
```yaml
networks:
  traefik:
    external: true
    name: traefik-network

services:
  your-app:
    image: ...
    networks:
      - traefik
```
- Để Traefik route traffic tới app container.

---

## 9. Reload Traefik
Traefik tự reload cấu hình động. Nếu cần ép reload:
```bash
docker-compose restart traefik
```

---

## 10. Kiểm tra SSL và routing
- Truy cập domain: https://example.com
- SSL hợp lệ (Let's Encrypt tự động)
- App hoạt động qua Traefik

---

## 11. Lệnh hữu ích
- **Xem log Traefik:**
  ```bash
  docker-compose logs -f traefik
  ```
- **Kiểm tra container đang chạy:**
  ```bash
  docker ps
  ```
- **Liệt kê network Docker:**
  ```bash
  docker network ls
  ```
- **Xem chi tiết network:**
  ```bash
  docker network inspect traefik-network
  ```

---

## 12. Lưu ý bảo mật
- Giữ file `letsencrypt/acme.json` an toàn (`chmod 600`).
- Không public dashboard Traefik nếu chưa bảo vệ bằng xác thực hoặc firewall.
- Thường xuyên cập nhật Traefik và container để vá lỗi bảo mật.

---

## 13. Tham khảo
- [Tài liệu Traefik](https://doc.traefik.io/traefik/)
- [Let's Encrypt](https://letsencrypt.org/)

---

Chúc bạn reverse proxy thành công! 