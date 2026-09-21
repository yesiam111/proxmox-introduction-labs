# Lab 02 (Guided) — Dựng Shared Storage: Cluster + NFS + Ceph

> Xây trạng thái nền cho các bài sau: hình thành Proxmox cluster 3 node, gắn NFS (ISO/backup), triển khai Ceph (MON/MGR/OSD), tạo pool RBD và đăng ký làm Proxmox storage. Sau bài này cụm có **shared storage** — điều kiện cho VM/CT (Bài 03), HA (Bài 05).
> **Chấm điểm:** không chấm điểm — cuối lab có khối lệnh xác nhận bắt buộc chạy sạch trước khi instructor snapshot **checkpoint 2**.

## Prerequisites

- Đã hoàn tất **guided Lab 01 + challenge 01** trên cả 3 node (checkpoint 1): 3 node cài sạch, hostname/hosts/repo/NTP đúng, **3 disk 25 GB trống hoàn toàn** trên mỗi node.
- Instructor đã dựng sẵn (Lab 00): **NFS server** trên jump host export `10.0.10.1:/srv/nfs/pve`.
- Quy hoạch network (4 NIC / 3 IP — bài này gán IP cho NIC 3 (Ceph) và NIC 4 (Corosync); NIC 2 để trống cho bond ở Bài 04):

| NIC | Mạng | Subnet | pve1 | pve2 | pve3 | MTU |
|-----|------|--------|------|------|------|-----|
| 1 | Management | 10.0.10.0/24 | .11 | .12 | .13 | 1500 (đã cấu hình Lab 01) |
| 3 | Ceph (public+cluster gộp) | 10.0.30.0/24 | .11 | .12 | .13 | **9000** (Bước 1) |
| 4 | Corosync | 10.0.20.0/24 | .11 | .12 | .13 | 1500 (Bước 1) |

---

## Bước 1 — Gán IP cho NIC Ceph và Corosync (mỗi node)

Installer Lab 01 chỉ cấu hình NIC quản trị (vmbr0). Gán IP tĩnh cho **NIC 3 (Ceph, MTU 9000)** và **NIC 4 (Corosync)**; **NIC 2 để trống** cho bond ở Bài 04. Tên NIC **có thể khác máy** — xác định trước:

```bash
ip -br a        # NIC1 mang IP 10.0.10.1N (vmbr0). Ghi lại tên NIC 3 (Ceph) và NIC 4 (Corosync)
```

Thêm cấu hình vào `/etc/network/interfaces` (đổi tên `nicXX` theo máy bạn; đổi `.11` theo node):

```bash
cat >> /etc/network/interfaces <<'EOF'

# Ceph public + cluster (gộp một đường) — jumbo 9000
auto nic2
iface nic2 inet static
    address 10.0.30.11/24
    mtu 9000

# Corosync ring (link0)
auto nic3
iface nic3 inet static
    address 10.0.20.11/24
EOF

ifreload -a
```

Xác minh (mỗi node):

```bash
ip -br a | grep -E '10\.0\.(20|30)'       # phải thấy 2 IP: ceph + corosync
ip link show nic2 | grep -o 'mtu 9000'   # Ceph NIC là jumbo
```

Kiểm tra chéo đường corosync và ceph — từ pve1 (kể cả jumbo end-to-end):

```bash
ping -c2 10.0.20.12 && ping -c2 10.0.20.13         # corosync tới pve2/pve3
ping -M do -s 8972 -c2 10.0.30.12                  # ceph jumbo 9000 tới pve2 (DF) — phải THÀNH CÔNG
ping -M do -s 8972 -c2 10.0.30.13                  # ceph jumbo 9000 tới pve3
```

> Nếu ping fail: kiểm tra tên NIC (`ip -br a`), typo IP, MTU chưa nhất quán, hoặc vSwitch lớp ngoài chưa cho phép mạng/jumbo. Sửa trước khi đi tiếp — cluster và Ceph phụ thuộc các đường này.

## Bước 2 — Hình thành Proxmox cluster (corosync trên link riêng)

Chỉ tạo cluster **một lần**, trên **pve1**. Đặt corosync ring lên mạng 10.0.20 (link riêng — best practice từ Bài 01):

```bash
# TRÊN pve1
pvecm create smartpro --link0 10.0.20.11
```

Thêm pve2 và pve3 vào cluster (chạy **trên từng node tương ứng**):

```bash
# TRÊN pve2
pvecm add pve1 --link0 10.0.20.12
# TRÊN pve3
pvecm add pve1 --link0 10.0.20.13
```

> `pvecm add pve1` kết nối tới node hiện có qua tên (mgmt để SSH/join); `--link0` khai địa chỉ corosync ring của node đang thêm. Gõ `yes` xác nhận SSH fingerprint khi được hỏi.

Xác minh (chạy trên bất kỳ node nào):

```bash
pvecm status                  # Quorate: Yes; Expected votes: 3; Total votes: 3
pvecm nodes                   # thấy pve1, pve2, pve3 — tất cả 'A' (available)
```

Kỳ vọng: **Quorate: Yes**, 3 node online. Từ đây `/etc/pve` đồng bộ trên cả 3 node — mọi thay đổi storage.cfg chỉ cần làm một lần.

## Bước 3 — Gắn NFS storage (ISO + backup)

NFS server đã được instructor dựng sẵn. Kiểm tra export nhìn thấy được, rồi đăng ký (chạy **một lần** trên pve1 — cả cluster nhận):

```bash
showmount -e 10.0.10.1                 # phải liệt kê /srv/nfs/pve
pvesm add nfs nfs-store --server 10.0.10.1 \
  --export /srv/nfs/pve --content iso,backup,vztmpl
pvesm status                           # 'nfs-store' phải 'active'
```

Kỳ vọng: `nfs-store` xuất hiện, cột Status = `active`, cột Type = `nfs`.

> Content types cố ý KHÔNG có `images` — export này để chứa ISO và backup, không phải disk VM. Disk VM sẽ nằm trên Ceph (Bước 6). Nếu `showmount` không ra export: lỗi ở NFS server/mạng, không phải PVE — báo instructor.

## Bước 4 — Cài Ceph và tạo MON/MGR (mỗi node)

Cài gói Ceph trên **cả 3 node**. Lab KHÔNG có subscription nên phải chỉ định repo no-subscription — `--repository` mặc định là `enterprise`, sẽ làm `apt` báo 401:

```bash
# TRÊN pve1, pve2, pve3
pveceph install --repository no-subscription
```

Khởi tạo cấu hình Ceph **một lần** trên pve1, khai mạng Ceph đã dựng ở Bước 1 (public+cluster gộp một đường):

```bash
# TRÊN pve1 — lab gộp public+cluster trên 10.0.30.0/24 (không tách --cluster-network)
pveceph init --network 10.0.30.0/24
```

> Production có thể tách `--cluster-network` (đường OSD↔OSD replication riêng) để recovery không đè client IO; lab dùng chung một đường jumbo cho gọn 4 NIC — nhắc trade-off khi giảng.

Tạo MON trên **cả 3 node** (MGR đi kèm MON đầu tiên):

```bash
# TRÊN pve1, rồi pve2, rồi pve3
pveceph mon create
```

Thêm một MGR standby cho dự phòng (trên pve2):

```bash
# TRÊN pve2
pveceph mgr create
```

Xác minh:

```bash
ceph -s
```

Kỳ vọng: `mon: 3 daemons, quorum pve1,pve2,pve3`; `mgr: pve1(active), standbys: pve2`; health có thể `HEALTH_WARN` vì **chưa có OSD** — bình thường ở bước này.

> Nếu thấy `HEALTH_WARN clock skew detected`: NTP chưa đồng bộ giữa các MON. Sửa ngay (nhắc Bài 01): cùng nguồn chrony + `chronyc makestep` trên node lệch. Ceph cảnh báo khi lệch > 0.05s.

## Bước 5 — Tạo OSD (BlueStore) từ disk trống

Mỗi disk 25 GB trống → 1 OSD. Xác định tên `by-id` của 3 disk trống trên từng node (KHÔNG dùng `/dev/sdX`):

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE          # 3 disk 25G, TYPE=disk, FSTYPE trống
ls -l /dev/disk/by-id/ | grep -v part   # lấy tên by-id ổn định
```

Tạo OSD cho **từng disk trống, trên từng node** (lặp 3 lần/node × 3 node = 9 OSD):

```bash
# TRÊN pve1 — lặp cho cả 3 disk by-id
pveceph osd create /dev/disk/by-id/ata-QEMU_HARDDISK_XXXX
# ... rồi disk 2, disk 3
# Lặp toàn bộ trên pve2 và pve3
```

> Web UI tương đương: Node → Ceph → OSD → Create OSD → chọn disk. Nếu báo disk không khả dụng: disk còn dính partition/filesystem cũ — `wipefs -a` và `sgdisk --zap-all` **đúng disk** (đối chiếu by-id) rồi thử lại. Đây chính là hệ quả của fault #6 trong challenge 01.

Xác minh:

```bash
ceph osd tree           # 3 host, mỗi host 3 osd; tất cả STATUS = up, REWEIGHT = 1
ceph -s                 # osd: 9 up, 9 in
```

Kỳ vọng: 9 OSD, tất cả `up`/`in`, phân bố đều 3/host.

## Bước 6 — Tạo pool RBD và đăng ký storage

Tạo pool replicated `vmpool` (size=3, min_size=2, autoscale on).
> **Bẫy:** `pveceph pool create` trên CLI mặc định `--add_storages 0` — pool được tạo nhưng **không** được
> đăng ký vào `storage.cfg`. Checkbox tương ứng trên GUI lại mặc định BẬT. Luôn truyền `--add_storages 1`
> khi dùng CLI, nếu không `pvesm status` sẽ không thấy `vmpool` và Bài 03/05 không cấp phát được.

```bash
# TRÊN pve1 (một lần cho cả cluster)
pveceph pool create vmpool --size 3 --min_size 2 --pg_autoscale_mode on --add_storages 1
```

Giới hạn RAM cho OSD — **bắt buộc trong lab nested** (node 10–12 GB RAM × 3 OSD, mặc định 4 GiB/OSD sẽ OOM):

```bash
ceph config set osd osd_memory_target 2147483648    # 2 GiB/OSD
```

Xác minh:

```bash
pvesm status                    # 'vmpool' (type rbd) phải 'active'
ceph osd pool ls detail         # vmpool: size 3, min_size 2
ceph df                         # thấy pool vmpool, %USED thấp
ceph -s                         # kỳ vọng HEALTH_OK
```

Kỳ vọng: `HEALTH_OK`, pool `vmpool` active. Đây là **shared storage** đầu tiên của cụm — disk VM (Bài 03) và HA (Bài 05) sẽ dùng nó.

> Kiểm chứng nhanh snapshot Ceph (tùy chọn, dọn sau): tạo 1 disk RBD nhỏ và snapshot để thấy Ceph snapshot tức thời — `rbd create vmpool/test --size 1G && rbd snap create vmpool/test@s1 && rbd snap ls vmpool/test && rbd snap purge vmpool/test && rbd rm vmpool/test` (phải `snap purge` trước — `rbd rm` từ chối image còn snapshot).

## Bước 7 — Xác nhận cuối (chạy trên pve1)

```bash
echo "== cluster =="
pvecm status | grep -E 'Quorate|Total votes'
echo "== nfs =="
pvesm status | awk '$1=="nfs-store"{print $1, $3}'
echo "== ceph health =="
ceph health
echo "== ceph mon/osd =="
ceph -s | grep -E 'mon:|osd:'
echo "== pool =="
pvesm status | awk '$1=="vmpool"{print $1, $2, $3}'
echo "== clock skew? =="
ceph health detail | grep -i skew || echo "no clock skew"
```

Kỳ vọng: `Quorate: Yes` · `nfs-store active` · `HEALTH_OK` · `mon: 3 ... quorum` · `osd: 9 up, 9 in` · `vmpool rbd active` · `no clock skew`.

**Báo instructor khi cả cụm HEALTH_OK** → instructor snapshot **checkpoint 2** → nhận challenge.

> Sau checkpoint 2, instructor mới chạy `setup02.sh` (dựng iSCSI target + tình huống challenge). Challenge chuyển sang backend **khác** (iSCSI + multipath) — không phải Ceph — để bạn tiếp xúc cả hai mô hình: "Proxmox quản lý redundancy" (Ceph, guided) và "array quản lý redundancy" (SAN, challenge).
