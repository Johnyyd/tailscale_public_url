# Tailscale Public URL Tunnel

Tạo public URL thông qua Tailscale Funnel để truy cập service trên máy host từ bất kỳ đâu. Hỗ trợ nhiều tunnel song song — lý tưởng cho home server.

## Prerequisites

- Docker & Docker Compose v2
- Tài khoản Tailscale (miễn phí): https://login.tailscale.com
- Các service đang chạy trên host (web server, app, API...)

## Bước 1: Tạo Auth Key

1. Đăng nhập https://login.tailscale.com
2. Vào **Settings** > **Keys**
3. Nhấn **Generate auth key**
4. Đánh dấu **Reusable** (dùng cho nhiều container)
5. Copy key (`tskey-auth-...`)

## Bước 2: Cấu hình Auth Key

Mở `docker-compose.yml`, thay thế giá trị `TS_AUTHKEY`:

```yaml
- TS_AUTHKEY=tskey-auth-XXXXXX-XXXXXXXX
```

## Bước 3: Tạo thư mục state

```bash
mkdir -p tunnels/{web,app,db,dashboard,api}
```

## Bước 4: Khởi động tất cả tunnel

```bash
docker compose up -d
```

Kiểm tra trạng thái:

```bash
docker compose ps
docker compose logs -f
```

Đợi đến khi tất cả container hiển thị `Bootstrapped` hoặc `Success`.

## Bước 5: Lấy Tailscale IP

```bash
docker exec tunnel-web tailscale ip -4
```

Mỗi container sẽ có cùng 1 Tailscale IP (vì dùng chung host network). Ghi lại IP `100.x.x.x`.

## Bước 6: Tạo Public URL cho từng tunnel

Thực hiện cho mỗi container, thay `<PORT>` bằng port service tương ứng:

```bash
# Tunnel 1: Web Server (port 80)
docker exec tunnel-web tailscale serve reset
docker exec tunnel-web tailscale funnel --bg http://100.x.x.x:80

# Tunnel 2: Application (port 3000)
docker exec tunnel-app tailscale serve reset
docker exec tunnel-app tailscale funnel --bg http://100.x.x.x:3000

# Tunnel 3: Database Admin (port 8080)
docker exec tunnel-db tailscale serve reset
docker exec tunnel-db tailscale funnel --bg http://100.x.x.x:8080

# Tunnel 4: Dashboard (port 3001)
docker exec tunnel-dashboard tailscale serve reset
docker exec tunnel-dashboard tailscale funnel --bg http://100.x.x.x:3001

# Tunnel 5: API (port 8000)
docker exec tunnel-api tailscale serve reset
docker exec tunnel-api tailscale funnel --bg http://100.x.x.x:8000
```

Output mỗi lệnh sẽ hiển thị public URL, ví dụ:

```
https://web.your-tailnet.ts.net/
https://app.your-tailnet.ts.net/
https://db.your-tailnet.ts.net/
https://dashboard.your-tailnet.ts.net/
https://api.your-tailnet.ts.net/
```

## Bước 7: Truy cập

Mở trình duyệt truy cập các URL trên. Có thể chia sẻ cho bất kỳ ai, không cần cài Tailscale.

---

## Quản lý nhiều tunnel

### Danh sách tunnel mẫu

| Container | Hostname | Service | Port | URL |
|-----------|----------|---------|------|-----|
| `tunnel-web` | `web` | Web Server | 80 | `https://web.your-tailnet.ts.net/` |
| `tunnel-app` | `app` | Application | 3000 | `https://app.your-tailnet.ts.net/` |
| `tunnel-db` | `db` | DB Admin | 8080 | `https://db.your-tailnet.ts.net/` |
| `tunnel-dashboard` | `dashboard` | Dashboard | 3001 | `https://dashboard.your-tailnet.ts.net/` |
| `tunnel-api` | `api` | API | 8000 | `https://api.your-tailnet.ts.net/` |

### Thêm tunnel mới

1. Copy một block service trong `docker-compose.yml`
2. Thay đổi `container_name`, `hostname`, và đường dẫn volume
3. Tạo thư mục state:

```bash
mkdir -p tunnels/<ten-moi>
```

4. Restart và tạo funnel:

```bash
docker compose up -d
docker exec tunnel-<ten-moi> tailscale serve reset
docker exec tunnel-<ten-moi> tailscale funnel --bg http://100.x.x.x:<PORT>
```

### Ví dụ home server

Giả sử bạn chạy các service sau trên host:

```
Nginx         → port 80
Next.js App   → port 3000
Home Assistant → port 8123
Grafana       → port 3000 (trên máy khác)
Syncthing     → port 8384
```

Cấu hình `docker-compose.yml`:

```yaml
services:
  tunnel-nginx:
    <<: *tailscale-common
    container_name: tunnel-nginx
    hostname: nginx
    volumes:
      - ./tunnels/nginx:/var/lib/tailscale

  tunnel-nextjs:
    <<: *tailscale-common
    container_name: tunnel-nextjs
    hostname: nextjs
    volumes:
      - ./tunnels/nextjs:/var/lib/tailscale

  tunnel-ha:
    <<: *tailscale-common
    container_name: tunnel-ha
    hostname: ha
    volumes:
      - ./tunnels/ha:/var/lib/tailscale
```

Tạo funnel:

```bash
docker compose up -d
docker exec tunnel-nginx tailscale funnel --bg http://100.x.x.x:80
docker exec tunnel-nextjs tailscale funnel --bg http://100.x.x.x:3000
docker exec tunnel-ha tailscale funnel --bg http://100.x.x.x:8123
```

---

## Lệnh thường dùng

```bash
# Khởi động tất cả
docker compose up -d

# Dừng tất cả
docker compose down

# Xem logs tất cả
docker compose logs -f

# Xem logs 1 container
docker exec tunnel-web tailscale status

# Lấy IP Tailscale
docker exec tunnel-web tailscale ip -4

# Reset funnel
docker exec tunnel-web tailscale serve reset

# Xem trạng thái serve/funnel
docker exec tunnel-web tailscale serve status
docker exec tunnel-web tailscale funnel status
```

---

## Lưu ý quan trọng

- **Tailscale IP có thể thay đổi** khi restart container. Luôn kiểm tra lại bằng `tailscale ip -4`.
- **Service phải lắng nghe trên `0.0.0.0`** hoặc `127.0.0.1` (với userspace) để Tailscale truy cập được.
- **Auth key có thể dùng chung** cho tất cả container nếu đánh dấu Reusable.
- **Mỗi container có state riêng** trong thư mục `tunnels/`, giúp giữ IP cố định.
- **Docker Compose v2**: dùng `docker compose`. **v1**: dùng `docker-compose`.

## Troubleshooting

| Vấn đề | Giải pháp |
|--------|-----------|
| Container không start | Kiểm tra auth key hợp lệ |
| `funnel` không hoạt động | Enable Tailscale Funnel trong Admin Console |
| Không truy cập được URL | Kiểm tra service đang chạy và đúng port |
| IP thay đổi sau restart | State lưu trong `tunnels/` sẽ giữ IP cũ |
| Port conflict | Đảm bảo mỗi service dùng port khác nhau |
| Container conflict hostname | Mỗi container phải có hostname riêng |

## Tham khảo

- [Tailscale Funnel](https://tailscale.com/kb/1248/tailscale-funnel)
- [Tailscale Serve](https://tailscale.com/kb/1242/tailscale-serve)
- [Tailscale Docker](https://tailscale.com/kb/1278/tailscale-docker)
