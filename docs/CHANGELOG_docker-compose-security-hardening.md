# Docker Compose Security Hardening — Changelog

> Base commit: `b1f6eadf` (dev branch)
> Date: 2025-04-17
> Scope: `docker-compose.yml`, `docker-compose.postgres.yml`, `docker-compose.selfservice.yml`

---

## Summary

Các thay đổi tập trung vào việc giảm attack surface bằng cách loại bỏ port binding trực tiếp ra host và chuẩn hóa port mapping cho nginx reverse proxy overlay.

---

## Changes by File

### 1. `docker-compose.yml` (Base service)

| # | Original | Changed | Security Rationale |
|---|----------|---------|-------------------|
| 1 | `ports: - "${GOCLAW_PORT:-18790}:18790"` | Commented out (`#ports`, `#  - "18790"`) | Loại bỏ host port binding mặc định. GoClaw API/UI chỉ accessible qua internal Docker network (`goclaw-net`), không expose trực tiếp ra host. Giảm attack surface — service chỉ reachable qua reverse proxy hoặc inter-container communication. |

**Tác động:**
- Container vẫn listen trên port 18790 nội bộ
- Không thể truy cập trực tiếp từ host machine qua `localhost:18790` trừ khi uncomment
- Buộc traffic đi qua nginx reverse proxy (`docker-compose.selfservice.yml`) hoặc truy cập qua Docker network

---

### 2. `docker-compose.postgres.yml` (PostgreSQL overlay)

| # | Original | Changed | Security Rationale |
|---|----------|---------|-------------------|
| 1 | `ports: - "${POSTGRES_PORT:-5432}:5432"` | Commented out (`#ports`, `#  - "5432"`) | PostgreSQL không còn expose port 5432 ra host. Database chỉ accessible từ các container trong cùng Docker network. Ngăn chặn unauthorized database access từ bên ngoài — tuân thủ principle of least privilege. |

**Tác động:**
- PostgreSQL chỉ reachable từ `goclaw` service qua internal DNS (`postgres:5432`)
- Không thể kết nối database từ host machine (ví dụ: pgAdmin, DBeaver) trừ khi uncomment
- Loại bỏ rủi ro brute-force password attack trên port 5432 từ network bên ngoài

---

### 3. `docker-compose.selfservice.yml` (Nginx UI overlay)

| # | Original | Changed | Security Rationale |
|---|----------|---------|-------------------|
| 1 | `ports: - "${GOCLAW_UI_PORT:-3000}:80"` | `ports: - "${GOCLAW_UI_PORT:-80}:80"` | Chuyển default port từ 3000 sang 80 (HTTP standard). Chuẩn hóa theo convention — port 80 là standard cho HTTP traffic. |
| 2 | _(không có)_ | Thêm `- "${GOCLAW_UI_PORT:-443}:443"` | Thêm HTTPS port mapping. Cho phép nginx serve SSL/TLS traffic trực tiếp — hỗ trợ SSL termination tại reverse proxy layer. |

**Tác động:**
- Nginx UI giờ listen trên cả port 80 (HTTP) và 443 (HTTPS)
- Hỗ trợ SSL termination — traffic được mã hóa từ client đến nginx
- Default port mapping chuẩn hơn cho production deployment

---

## Security Model After Changes

```
┌─────────────────────────────────────────────────┐
│  Host Machine                                   │
│                                                 │
│  Port 80/443 ──► nginx (goclaw-ui)              │
│                    │                             │
│  ┌─────────────────┼──── goclaw-net ───────────┐│
│  │                 ▼                            ││
│  │  goclaw:18790 (internal only)                ││
│  │       │                                      ││
│  │       ▼                                      ││
│  │  postgres:5432 (internal only)               ││
│  └──────────────────────────────────────────────┘│
│                                                 │
│  ✗ localhost:18790  (blocked — no host binding) │
│  ✗ localhost:5432   (blocked — no host binding) │
│  ✓ localhost:80     (via nginx)                 │
│  ✓ localhost:443    (via nginx + SSL)           │
└─────────────────────────────────────────────────┘
```

---

## Pre-existing Security Controls (unchanged)

Các security controls sau đã có sẵn trong bản gốc và được giữ nguyên:

| Control | File | Detail |
|---------|------|--------|
| `security_opt: no-new-privileges` | `docker-compose.yml` | Ngăn privilege escalation trong container |
| `cap_drop: ALL` + selective `cap_add` | `docker-compose.yml` | Drop tất cả Linux capabilities, chỉ add SETUID/SETGID/CHOWN |
| `tmpfs: /tmp:rw,noexec,nosuid` | `docker-compose.yml` | Mount /tmp với noexec — ngăn execute binary từ tmp |
| Resource limits (1G RAM, 2 CPU, 200 PIDs) | `docker-compose.yml` | Giới hạn resource — chống DoS/fork bomb |
| `init: true` | `docker-compose.yml` | Proper PID 1 handling — tránh zombie processes |
| `.env required: false` | `docker-compose.yml` | Graceful fallback khi không có .env file |

---

## Recommendations (chưa thực hiện)

| # | Recommendation | Priority | Rationale |
|---|---------------|----------|-----------|
| 1 | Thêm `read_only: true` cho PostgreSQL container | Medium | Filesystem read-only trừ volumes — giảm persistence attack |
| 2 | Thêm `security_opt`, `cap_drop` cho PostgreSQL | Medium | PostgreSQL container chưa có hardening tương đương goclaw |
| 3 | Thêm `security_opt`, `cap_drop` cho nginx (goclaw-ui) | Medium | Nginx container chưa có security controls |
| 4 | Network isolation: tách `postgres` vào network riêng | Low | PostgreSQL chỉ cần communicate với goclaw, không cần default network |
| 5 | Thêm healthcheck cho `goclaw-ui` | Low | Detect nginx failure sớm hơn |
| 6 | Pin image tags thay vì dùng `:latest` | Low | Reproducible builds, tránh supply chain risk |

---

## Note về `GOCLAW_UI_PORT` variable

Hiện tại cả port 80 và 443 đều dùng cùng biến `${GOCLAW_UI_PORT}`. Nếu user set `GOCLAW_UI_PORT=8080`, cả hai mapping sẽ thành `8080:80` và `8080:443` — gây conflict. Nên tách thành hai biến riêng:

```yaml
ports:
  - "${GOCLAW_UI_HTTP_PORT:-80}:80"
  - "${GOCLAW_UI_HTTPS_PORT:-443}:443"
```


---

## Phase 2 — Volume Bind Mounts & Nginx SSL Termination

> Date: 2025-04-17

### Summary

Chuyển tất cả Docker named volumes sang host bind mounts để dễ quản lý data trên host, và thêm nginx reverse proxy với SSL termination đầy đủ.

---

### 4. Volume Migration: Named Volumes → Host Bind Mounts

Tất cả named volumes đã được chuyển sang bind mount với tên thư mục trùng volume name gốc:

| File | Original (Named Volume) | Changed (Bind Mount) | Container Path |
|------|------------------------|---------------------|----------------|
| `docker-compose.yml` | `goclaw-data:` | `./goclaw-data:` | `/app/data` |
| `docker-compose.yml` | `goclaw-workspace:` | `./goclaw-workspace:` | `/app/workspace` |
| `docker-compose.postgres.yml` | `postgres-data:` | `./postgres-data:` | `/var/lib/postgresql` |
| `docker-compose.postgres.yml` | `goclaw-skills:` | `./goclaw-skills:` | `/app/skills` |
| `docker-compose.postgres.yml` | `goclaw-workspace:` | `./goclaw-workspace:` | `/app/.goclaw` |

Các top-level `volumes:` declarations đã được xóa vì không còn cần thiết.

**Lợi ích:**
- Data nằm trực tiếp trên host filesystem, dễ backup/restore
- Dễ inspect và debug data mà không cần `docker volume inspect`
- Portable — copy thư mục là có toàn bộ data

**Lưu ý:**
- Cần đảm bảo thư mục host tồn tại trước khi `docker compose up`
- Permission trên host directory phải phù hợp với UID/GID trong container

---

### 5. Nginx SSL Termination (`docker-compose.selfservice.yml`)

| # | Original | Changed | Rationale |
|---|----------|---------|-----------|
| 1 | `image: ghcr.io/nextlevelbuilder/goclaw-web:latest` | `image: nginx:stable-alpine` | Dùng official nginx image — lightweight, trusted, dễ maintain |
| 2 | `${GOCLAW_UI_PORT:-80}:80` + `${GOCLAW_UI_PORT:-443}:443` | `${GOCLAW_UI_HTTP_PORT:-80}:80` + `${GOCLAW_UI_HTTPS_PORT:-443}:443` | Tách biến HTTP/HTTPS port — fix bug port conflict khi override |
| 3 | _(không có volumes)_ | Mount `nginx.conf`, `conf.d/`, `ssl/`, `logs/nginx` | Bind mount config + certs + logs ra host |

**Cấu trúc nginx mới:**

```
goclaw/
├── nginx/
│   ├── nginx.conf              # Main nginx config (server_tokens off, gzip, rate limiting)
│   ├── conf.d/
│   │   ├── default.conf        # Virtual host: HTTP→HTTPS redirect + reverse proxy to goclaw:18790
│   │   └── mod_protect.conf    # Security headers (HSTS, XSS, nosniff) + TLS 1.2/1.3 + cipher suite
│   └── ssl/
│       ├── README.md           # Instructions for placing certs
│       ├── fullchain.pem       # (user-provided) SSL certificate chain
│       └── private.key         # (user-provided) Private key — DO NOT commit
├── logs/
│   └── nginx/                  # Nginx access + error logs (bind mounted)
```

**Security features trong nginx config:**
- HTTP → HTTPS 301 redirect
- HSTS với `max-age=31536000; includeSubDomains; preload`
- X-XSS-Protection, X-Content-Type-Options, X-Frame-Options headers
- TLS 1.2/1.3 only (no SSLv3, TLS 1.0/1.1)
- Modern cipher suite (no RC4, 3DES, MD5, NULL)
- `server_tokens off` — ẩn nginx version
- Rate limiting: 20 req/s per IP
- WebSocket proxy support (`/ws` endpoint)
- Hidden file access denied (`location ~ /\.`)
- All config files mounted `:ro` (read-only)


---

### 6. Fix: Nginx `server_name` directive

| # | Original | Changed | Rationale |
|---|----------|---------|-----------|
| 1 | `server_name ${GOCLAW_SERVER_NAME}` | `server_name _` | Nginx không tự resolve env vars trong config file. Dùng `_` (wildcard catch-all) làm default. Thay bằng domain thực khi deploy production. |
| 2 | `return 301 https://$server_name$request_uri` | `return 301 https://$host$request_uri` | Dùng `$host` thay vì `$server_name` để redirect chính xác theo hostname client gửi đến. |

**Cách thay đổi domain khi deploy:**
1. Sửa `server_name _` thành `server_name goclaw.yourdomain.com` trong `nginx/conf.d/default.conf`
2. Reload nginx: `docker exec goclaw-ui nginx -s reload`
