# Immich -> Proxmox HDD hdd-backup-pt (Portainer 18892)

Mục tiêu: ảnh/video và tất cả media Immich trên HDD, PostgreSQL giữ nguyên SSD; không đổi cổng 18892 và không thay Stack khác.

KHÔNG đổi đường dẫn upload trong Portainer cho tới khi ổ HDD đã được mount bên trong Docker VM/LXC và dữ liệu cũ đã sao chép. Tên storage hdd-backup-pt chỉ là ID trong Proxmox, KHÔNG phải đường dẫn Docker. Chưa xác nhận Docker chạy VM hay LXC; phải xác minh.

## BƯỚC 1. Kiểm tra storage và Docker guest trên Proxmox

Chạy trên Proxmox HOST các lệnh chỉ đọc:

    pvesm status
    grep -A 8 -B 1 'hdd-backup-pt' /etc/pve/storage.cfg
    qm list
    pct list
    lsblk -f

- storage dạng pbs (Proxmox Backup Server) KHÔNG phải thư mục ảnh Immich.
- storage chỉ có content backup thì KHÔNG thể chọn làm đĩa VM nếu không có hỗ trợ content images. Không tự sửa storage content khi chưa kiểm tra dung lượng và cấu trúc.
- Phải xác nhận Docker đang chạy VM nào (qm) hay LXC nào (pct).

## BƯỚC 2A. Nếu Docker nằm trong VM

Proxmox GUI: VM Docker > Hardware > Add > Hard Disk > Storage chọn hdd-backup-pt nếu có hỗ trợ Disk image. Tạo một đĩa ẢO MỚI dành riêng cho Immich; không xóa/format ổ cũ.

Bên trong VM Docker:

    lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,UUID,MODEL

Chỉ tạo phân vùng / filesystem cho đĩa MỚI và chắc chắn TRỐNG. Không chạy mkfs trên /dev/sdX khi chưa xác minh; mkfs xóa dữ liệu. Mount vào /mnt/immich-hdd, thêm UUID thật vào /etc/fstab để tự gắn sau reboot. Ví dụ minh họa dòng fstab (KHÔNG DÙNG nguyên xi):

    UUID=<UUID-phân-vùng-mới> /mnt/immich-hdd ext4 defaults 0 2

Kiểm tra trong máy Docker:

    mountpoint -q /mnt/immich-hdd && df -hT /mnt/immich-hdd

Nếu storage không hỗ trợ images, không thể bind trực tiếp Proxmox host path vào VM chỉ bằng sửa Compose; phải chọn hình thức gắn storage phù hợp khác.

## BƯỚC 2B. Nếu Docker nằm trong LXC

Nếu Proxmox storage là Directory, lấy đường dẫn path THỰC từ /etc/pve/storage.cfg. Trên Proxmox host, tạo thư mục media riêng nằm trong ổ đã gắn:

    mkdir -p /DUONG-DAN-THUC-CUA-HDD/immich-media

Xem cấu hình CT:

    pct config <CTID>

Chỉ khi chắc chắn mp0 chưa dùng, bind mount vào CT:

    pct set <CTID> -mp0 /DUONG-DAN-THUC-CUA-HDD/immich-media,mp=/mnt/immich-hdd

Nếu mp0 đã dùng, chọn mp phù hợp khác. Khởi động lại LXC nếu cần. Với unprivileged LXC có thể cần xử lý UID/GID mapping; không dùng chmod 777 bừa bãi.

Bên trong Docker LXC:

    mountpoint -q /mnt/immich-hdd && df -hT /mnt/immich-hdd

## BƯỚC 3. Chuyển file media đang có, chỉ chạy trong máy Docker

Trước khi chuyển, sao lưu database và media. Tắt backup tự động trên điện thoại để không phát sinh upload. Trên máy Docker:

    mountpoint -q /mnt/immich-hdd || { echo "HDD CHUA MOUNT - DUNG LAI"; exit 1; }
    docker ps --format 'table {{.Names}}\t{{.Ports}}'
    docker inspect immich-immich-server-1 --format '{{range .Mounts}}{{if eq .Destination "/data"}}{{.Source}}{{end}}{{end}}'

Nhìn đường dẫn được trả về. File Compose hiện mặc định là /srv/immich-18892/library. Nếu inspect trả về khác thì PHẢI đổi SOURCE trong lệnh rsync dưới đây cho khớp. Không sao chép từ một đường dẫn đoán mò.

    sudo mkdir -p /mnt/immich-hdd/immich-library
    docker stop immich-immich-server-1
    mountpoint -q /mnt/immich-hdd || exit 1
    sudo rsync -aHAX --numeric-ids --info=progress2 /srv/immich-18892/library/ /mnt/immich-hdd/immich-library/
    sudo du -sh /srv/immich-18892/library /mnt/immich-hdd/immich-library

Dừng duy nhất Immich Server; không xóa volume/PostgreSQL; không xóa thư mục cũ. Nếu ứng dụng hiện đã có lỗi unhealthy / connection refused thì cần kiểm tra lỗi đó sau khi dữ liệu chuyển thành công, không coi đổi ổ là cách chữa lỗi web.

## BƯỚC 4. Portainer: thay một biến môi trường duy nhất

Stack Immich HIỆN TẠI > Environment variables:

    IMMICH_UPLOAD_LOCATION=/mnt/immich-hdd/immich-library

Giữ nguyên:
- IMMICH_DB_DATA_LOCATION=/srv/immich-18892/postgres (nếu chưa từng thay)
- IMMICH_DB_PASSWORD (mật khẩu đang dùng)
- IMMICH_BIND_IP (IP thực tế), cổng host 18892.

Pull and redeploy, dùng:
- Repository: https://github.com/hoangthang0106/immich
- Branch: refs/heads/portainer-18892
- Compose path: docker-compose.portainer.yml

Kiểm tra Docker Logs, mounts của /data, mở thử nhiều ảnh/video. Chỉ xóa nguồn cũ khi đã xác minh ảnh, metadata, thumbnail và đã có backup riêng độc lập.

## CẢNH BÁO

- Trước mỗi lần reboot/redeploy phải đảm bảo /mnt/immich-hdd là mount thật. Docker bind mount có thể tự tạo thư mục trên SSD nếu ổ vắng mặt, khiến Immich trông như mất ảnh.
- HDD hdd-backup-pt chứa ảnh chính KHÔNG tự thành bản backup! Ảnh và PostgreSQL cần sao lưu ra ổ/máy khác. Không dùng chung HDD như bản chính và bản dự phòng duy nhất.
- Không chuyển PostgreSQL sang SMB/NFS hoặc ổ backup mạng. PostgreSQL trên SSD cục bộ.
- Repo fork hiện Public. KHÔNG commit mật khẩu, token, IP nội bộ hay mountpoint chứa thông tin nhạy cảm.
- Mặc định Git Stack đã có biến IMMICH_UPLOAD_LOCATION; hiện KHÔNG bắt buộc thay Compose, chỉ cần Portainer env sau khi mount đúng.

Nguồn:
https://docs.immich.app/install/docker-compose/
https://docs.immich.app/administration/backup-and-restore/
https://github.com/proxmox/pve-docs/blob/master/pct.adoc
