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
git clone https://github.com/thienphuc2611/traefik-reverse-proxy-with-docker-compose.git
cd traefik-reverse-proxy-with-docker-compose
# (Tuỳ chọn) Xoá thư mục .git để source sạch
rm -rf .git
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
   cp dynamic/sites/_template-site.yml dynamic/sites/tenmiencuaban.com.yml
   ```
2. **Đổi tên và chỉnh sửa file `tenmiencuaban.com.yml`:**
   - Đổi tên file thành tên miền bạn muốn cấu hình (ví dụ: `tenmiencuaban.com.yml`).
   - Mở file và thay các phần sau:
     - `<router_name>`: Đặt tên router, nên đặt giống tên miền (không dấu cách).
     - `<domain>`: Thay bằng tên miền thật, ví dụ: `tenmiencuaban.com`.
     - `<service_name>`: Đặt tên service, nên đặt giống tên miền (không dấu cách).
     - `<container_name>`: Tên container ứng dụng backend của bạn (ví dụ: `tenmiencuaban-nginx`).
   - Nếu muốn redirect www về non-www, giữ nguyên router www như template.
   - Nếu không dùng www, có thể xoá router www.
   - Ví dụ sau khi chỉnh sửa:
     ```yaml
     http:
       routers:
         tenmiencuaban:
           rule: "Host(`tenmiencuaban.com`)"
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
           service: tenmiencuaban

         tenmiencuaban_www:
           rule: "Host(`www.tenmiencuaban.com`)"
           entryPoints:
             - web
             - websecure
           tls:
             certResolver: letsencrypt
           middlewares:
             - redirect-www-to-root@file
             - redirect-https@file
           service: noop@internal

       services:
         tenmiencuaban:
           loadBalancer:
             servers:
               - url: "http://tenmiencuaban-nginx:80"
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

---

## Hỏi & Đáp

**Hỏi: Tôi đã cấu hình chuyển hướng www về non-www nhưng vẫn không hoạt động, tại sao?**

**Trả lời:**
- Hãy chắc chắn bạn đã tạo bản ghi DNS (A hoặc CNAME) cho `www.tenmiencuaban.com` trỏ về đúng IP server.
- Nếu bản ghi DNS cho `www` chưa tồn tại hoặc trỏ sai, mọi truy cập vào `www.tenmiencuaban.com` sẽ không tới được Traefik để thực hiện chuyển hướng.
- Vào trang quản lý DNS của nhà cung cấp domain và thêm/cập nhật bản ghi `www` trỏ về server.
- **Nên tạo bản ghi nào?**
  - Nếu server của bạn có IP tĩnh, hãy tạo bản ghi **A**:
    - `www` → `IP_SERVER_CỦA_BẠN`
  - Nếu muốn trỏ `www` về domain gốc, hãy tạo bản ghi **CNAME**:
    - `www` → `tenmiencuaban.com`
- Đợi vài phút để DNS cập nhật rồi thử lại.

--- 