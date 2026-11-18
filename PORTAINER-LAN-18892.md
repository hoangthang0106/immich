# Immich Private LAN — Git Stack Portainer, cổng 18892

**Mục đích:** Portainer đọc trực tiếp file `docker-compose.portainer.yml` trong fork `hoangthang0106/immich`. Sau khi chỉnh file trên GitHub, chỉ cần **Pull and redeploy** trong Portainer. Không cần dán lại YAML vào Web editor.

## Cài lần đầu

1. Trên Docker VM/LXC (không phải Proxmox hypervisor), xác minh port **18892 chưa bị dùng**: `sudo ss -lntp | grep ':18892'` và `docker ps --format 'table {{.Names}}\t{{.Ports}}'`. Không deploy đè nếu có ứng dụng đang chiếm port.
2. Tạo đường dẫn lưu trữ bền vững trên host: `sudo mkdir -p /srv/immich-18892/library /srv/immich-18892/postgres`. **PostgreSQL phải nằm trên SSD cục bộ**, không đặt trên SMB/NFS NAS.
3. Portainer → **Stacks → Add stack → Repository / Git repository**:
   - **Name:** `immich-private-18892`
   - **Repository URL:** `https://github.com/hoangthang0106/immich`
   - **Repository reference:** `refs/heads/main`
   - **Compose path:** `docker-compose.portainer.yml`
4. Trong **Environment variables** của Stack, thiết lập duy nhất 2 biến bắt buộc:
   - `IMMICH_BIND_IP`: IP LAN thực tế của Docker host, không dùng `0.0.0.0`. Chỉ gán IP đã kiểm tra có trên interface của VM/LXC.
   - `IMMICH_DB_PASSWORD`: mật khẩu mạnh, **chỉ chữ/số**, tạo bằng `openssl rand -hex 24`; giữ bí mật trong Portainer, không đưa lên GitHub.
   - Có thể thay đường dẫn mặc định bằng `IMMICH_UPLOAD_LOCATION` và `IMMICH_DB_DATA_LOCATION`.
5. Deploy → sau khi Docker khởi động ổn định, mở `http://<IMMICH_BIND_IP>:18892` trong LAN được firewall cho phép. Thiết lập tài khoản quản trị.

## Cập nhật về sau

1. Chỉnh `docker-compose.portainer.yml` trong fork và commit lên `main`.
2. Portainer → Stack `immich-private-18892` → **Pull and redeploy**. Portainer có thể hỗ trợ GitOps polling tùy phiên bản nhưng **không bật cập nhật Immich tự động không kiểm soát**.
3. Trước khi nâng cấp Immich, **sao lưu database và thư viện ảnh**. Nâng `immich-server` và `immich-machine-learning` lên cùng tag stable mới. Xem release notes chính thức để biết yêu cầu migration.
4. Không xóa thư mục `/srv/immich-18892` hoặc volumes khi redeploy. Nếu Immich đã chạy ở port 18892 từ bản cài cũ, **không tạo Stack thứ hai** dùng chung port và dữ liệu.
5. Nếu database đã được khởi tạo bằng bản Web editor trước, `IMMICH_DB_PASSWORD` phải khớp mật khẩu DB hiện có. Không đăng mật khẩu đó lên GitHub.

## Cảnh báo bảo mật

- Fork GitHub hiện vẫn **PUBLIC**. File này không chứa IP LAN thật, mật khẩu, token hay thông tin cá nhân. Muốn chỉ mình bạn xem repo, cần tách fork khỏi fork network Public rồi đổi quyền Private bằng GitHub Settings. Kết nối GitHub hiện tại không hỗ trợ thao tác đổi quyền.
- Khi repository trở thành **Private**, bật xác thực Git trong Portainer bằng token chỉ có quyền đọc repo đó; nếu không Portainer sẽ không cập nhật được.
- Cổng duy nhất publish trên Docker host là **18892**. PostgreSQL, Valkey và Machine Learning không mở cổng ra LAN.
- Chỉ bind IP LAN **không thay thế firewall**. Không port-forward/NAT ra Internet; giới hạn IP/VLAN được phép; dùng HTTPS qua Caddy hoặc VPN nếu truy cập từ bên ngoài.
- HTTP LAN chưa mã hóa; tránh sử dụng mạng không tin cậy cho đăng nhập và ảnh riêng tư.
- Image tag ghim `v3.4.1`, Valkey/Postgres digest theo phát hành ổn định. Không tự build source từ `main`, vốn là nhánh phát triển.
- Không có kết nối trực tiếp tới Proxmox/Portainer nên chưa xác minh host IP, port trống hoặc chạy thử Docker thực tế.

## Tài liệu chính thức

- Cài Docker Compose: https://docs.immich.app/install/docker-compose/
- Cài Portainer: https://docs.immich.app/install/portainer/
- GitHub releases: https://github.com/immich-app/immich/releases
