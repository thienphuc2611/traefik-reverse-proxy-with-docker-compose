# Traefik Reverse Proxy with Docker Compose

> Instead of using Docker labels for each container, this project leverages file-based configuration for Traefik. This approach is easier to maintain, does not require container restarts when updating routing or middleware, and is especially suitable for servers running multiple Docker Compose projects. It also provides better security and separation of concerns.

This guide will help you deploy a production-ready Traefik v3 reverse proxy using Docker Compose. It covers automatic SSL with Let's Encrypt, dynamic configuration, and how to add new sites and applications.

---

**Why file-based configuration?**
- Easier to maintain than Docker labels
- No need to restart containers when updating routing or middleware
- Suitable for servers running multiple Docker Compose projects
- Provides better security and separation of concerns

---

## Usage Scenarios

You can deploy Traefik in two main ways, depending on your needs:

### 1. Standard Reverse Proxy (for your apps)
- Use `docker-compose.yml` to run Traefik as a reverse proxy for your applications.
- Exposes ports 80, 443, and 443/udp for HTTP/HTTPS traffic.
- Suitable for most use cases: your apps are accessible via your domain or server IP.
- Start with:
  ```bash
  docker-compose up -d
  ```

### 2. Dashboard on a Separate Subdomain
- Use `docker-compose.dashboard.yml` if you want the Traefik dashboard accessible via a dedicated subdomain (e.g., traefik.yourdomain.com).
- Exposes ports 80, 443, and 443/udp (no need to expose 3080:8080).
- Update the subdomain in `dynamic/sites/traefik-dashboard.yml` and point your DNS accordingly.
- Start with:
  ```bash
  docker-compose -f docker-compose.dashboard.yml up -d
  ```

---

## Directory Structure

```
traefik/
├── docker-compose.yml               # Main Docker Compose file for Traefik (reverse proxy for apps)
├── docker-compose.dashboard.yml     # Docker Compose file for Traefik dashboard on a subdomain
├── traefik.yml                      # Static Traefik configuration
├── letsencrypt/                     # Stores SSL certificates (acme.json)
│   └── acme.json
├── dynamic/                         # Dynamic configuration directory
│   ├── middlewares/                 # Common middlewares (gzip, rate-limit, etc.)
│   │   ├── cache-static.yml
│   │   ├── error-pages.yml
│   │   ├── gzip.yml
│   │   ├── rate-limit.yml
│   │   ├── redirect-https.yml
│   │   ├── secure-headers.yml
│   │   └── whitelist.yml
│   └── sites/                       # Per-site configuration files
│       ├── _template-site.yml
│       └── traefik-dashboard.yml    # Router config for Traefik dashboard subdomain
└── README.md                        # Project documentation
```

---

## Prerequisites
- Docker & Docker Compose installed
- A domain name pointed to your server's public IP

---

## 1. Clone the Repository
```bash
gh repo clone thienphuc2611/traefik-reverse-proxy-with-docker-compose
cd traefik-reverse-proxy-with-docker-compose
```

---

## 2. (Optional) Rename the directory
If you want to use a shorter directory name:
```bash
mv traefik-reverse-proxy-with-docker-compose traefik
cd traefik
```

---

## 3. Configure Traefik
- Open `docker-compose.yml` and set your email for Let's Encrypt in the `command:` section:
  ```yaml
  - --certificatesresolvers.letsencrypt.acme.email=your@email.com
  ```
- (Optional) Edit `traefik.yml` for advanced static settings.

---

## 4. Prepare SSL Storage
```bash
mkdir -p letsencrypt
chmod 600 letsencrypt/acme.json || touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json
```

---

## 5. Create Traefik Network
If you use an external network in your `docker-compose.yml`, you must create it before starting Traefik:
```bash
docker network create traefik-network
```

---

## 6. Start Traefik
```bash
docker-compose up -d
```
- Traefik will listen on ports 80 (HTTP) and 443 (HTTPS).
- The dashboard is available at https://<your-domain>:8080 (if enabled).

---

## 7. Add a New Site
1. **Copy the template:**
   ```bash
   cp dynamic/sites/_template-site.yml dynamic/sites/example.com.yml
   ```
2. **Edit `example.com.yml`:**
   - Set `rule: "Host(`example.com`)"`
   - Set the correct service name and container name
   - Example:
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
3. **Ensure your app container is on the same Docker network as Traefik** (see next step).

---

## 8. Connect Your App to the Traefik Network
In your app's `docker-compose.yml`, add the following:
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
- This allows Traefik to route traffic to your app container.

---

## 9. Reload Traefik
Traefik automatically reloads dynamic configuration. If you need to force reload:
```bash
docker-compose restart traefik
```

---

## 10. Check SSL and Routing
- Visit your domain: https://example.com
- SSL should be valid (auto-generated by Let's Encrypt)
- Your app should be accessible via Traefik

---

## 11. Useful Commands
- **View Traefik logs:**
  ```bash
  docker-compose logs -f traefik
  ```
- **Check running containers:**
  ```bash
  docker ps
  ```
- **List Docker networks:**
  ```bash
  docker network ls
  ```
- **Inspect a network:**
  ```bash
  docker network inspect traefik-network
  ```

---

## 12. Security Notes
- Keep your `letsencrypt/acme.json` file secure (`chmod 600`).
- Do not expose the Traefik dashboard publicly unless you secure it with authentication or firewall rules.
- Regularly update Traefik and your containers for security patches.

---

## 13. References
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Let's Encrypt](https://letsencrypt.org/)

---

Happy reverse proxying! 

--- 