# Lab 03 (Guided) — VM, Container, Template và Clone

> Xây trạng thái nền cho các bài sau trên **shared storage** đã dựng ở Bài 02: một VM Linux chuẩn (VirtIO + guest agent), một VM Template + clone, và một LXC unprivileged có mountpoint. Bài 04 sẽ nối các máy này vào VLAN/firewall/RBAC.
> **Chấm điểm:** không chấm điểm — cuối lab có khối lệnh xác nhận bắt buộc chạy sạch trước khi instructor snapshot **checkpoint 3**.

## Prerequisites

- Đã hoàn tất **guided + challenge Bài 02** (checkpoint 2): cluster quorate, Ceph `HEALTH_OK`, `vmpool` (RBD) và `nfs-store` (NFS) đều `active`.
- `nfs-store` có content `iso` và `vztmpl` (chứa ISO và CT template).
- Máy có kết nối tải cloud image + CT template (qua repo/mirror mà lớp cấp).

## Quy ước VMID trong lab này

| Tài nguyên | VMID |
|-----------|------|
| VM Template | 9000 |
| Full clone (web-01 — VM nền dùng ở Bài 04/05) | 130 |
| Linked clone (web-02) | 132 |
| LXC unprivileged | 200 |

> Challenge Bài 03 dùng **VMID 320** (dải riêng) — không đụng dải guided.

---

## Bước 1 — Tải cloud image và tạo VM Template (trên pve1)

Dùng cloud image Debian + Cloud-Init để tạo VM nhanh, không cài từ ISO. Lấy image từ **mirror của lớp** (instructor stage sẵn trong Lab 00 tại `nfs-store`/mirror nội bộ — không phụ thuộc internet ngoài):

```bash
# Cách 1 (khuyến nghị): copy từ mirror nội bộ lớp đã có sẵn
cp /mnt/pve/nfs-store/images/debian-13-generic-amd64.qcow2 /root/

# Cách 2 (nếu lớp cho phép internet): tải bản 'latest' từ upstream
cd /root && wget -O debian-13-generic-amd64.qcow2 \
  https://cloud.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2
# LƯU Ý: cloud image của Debian KHÔNG kèm qemu-guest-agent — ta sẽ tự cài trong guest ở Bước 1 (dưới).
```

Tạo khung VM 9000 và import đĩa cloud image vào `vmpool` (Ceph):

```bash
qm create 9000 --name tmpl-debian13 --memory 2048 --cores 2 \
  --machine q35 --bios ovmf --cpu x86-64-v2-AES \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-single
qm set 9000 --efidisk0 vmpool:0,efitype=4m           # EFI disk cho OVMF
qm disk import 9000 debian-13-generic-amd64.qcow2 vmpool
qm set 9000 --scsi0 vmpool:vm-9000-disk-1,discard=on,ssd=1
qm set 9000 --boot order=scsi0
qm set 9000 --ide2 vmpool:cloudinit                  # CloudInit drive
qm set 9000 --serial0 socket --vga serial0           # console cho cloud image
qm set 9000 --agent 1                                # MỞ KÊNH agent (chưa cài gì trong guest)
```

> Số thứ tự disk sau `qm disk import` có thể là `vm-9000-disk-1` (vì `disk-0` là efidisk). Kiểm tra bằng `qm config 9000` và điều chỉnh tên cho khớp trước khi gán `--scsi0`.

Đặt tham số Cloud-Init:

```bash
qm set 9000 --ciuser labadmin --cipassword 'Lab@2026' \
  --sshkey ~/.ssh/id_rsa.pub \
  --nameserver 10.0.10.1 --searchdomain lab.local
```

### Boot một lần để cài guest agent — TRƯỚC khi convert sang template

**Cloud image chính thức của Debian KHÔNG kèm `qemu-guest-agent`.** Cờ `--agent 1` ở trên chỉ mở
**kênh** virtio-serial phía hypervisor — nó không cài gì vào trong guest. Nếu convert template ngay
bây giờ thì mọi clone sau này đều thiếu agent, và `qm agent ping` ở Bước 3 sẽ fail.

Gắn IP tạm để VM ra được Internet qua NAT trên jump, boot lên và cài:

```bash
qm set 9000 --ipconfig0 ip=10.0.10.39/24,gw=10.0.10.1   # IP TẠM — ngoài dải VM của lab
qm start 9000
qm terminal 9000                 # user labadmin / Lab@2026 ; thoát console: Ctrl+O
```

Trong guest:

```bash
ip -br a                         # cloud-init đã gán 10.0.10.39 chưa
sudo apt-get update              # nếu treo ở đây: NAT/DNS của lab hỏng, không phải lỗi VM
sudo apt-get install -y qemu-guest-agent
systemctl is-active qemu-guest-agent         # phải in 'active' — KHÔNG cần enable, xem ghi chú dưới
# nếu chưa active (udev chưa kịp bắt cổng virtio):
sudo systemctl start qemu-guest-agent

# Dọn dẹp cho Golden Image — BẮT BUỘC trước khi làm template:
sudo cloud-init clean --logs     # xoá state first-boot để mỗi clone chạy lại cloud-init
sudo truncate -s 0 /etc/machine-id   # machine-id phải là duy nhất cho từng clone
sudo poweroff
```

Chờ VM tắt hẳn rồi gỡ IP tạm và convert:

```bash
qm status 9000                   # đợi tới khi 'stopped'
qm set 9000 --delete ipconfig0   # template KHÔNG mang IP — mỗi clone gán riêng (Bước 2)
qm template 9000
qm config 9000 | grep -E 'template|scsi0|agent'      # template: 1
```

Kỳ vọng: `qm config 9000` có dòng `template: 1`, disk trên `vmpool`, `agent: 1`, **không còn** `ipconfig0`.

> **Không `systemctl enable qemu-guest-agent`** — unit này không có `[Install]`, nó do udev bật khi
> thấy cổng `/dev/virtio-ports/org.qemu.guest_agent.0`. Đọc `systemctl is-enabled`:
>
> | Kết quả | Nghĩa là | Làm gì |
> |---|---|---|
> | `static` | Đã cài, chờ udev kích hoạt — **đúng** | kiểm tiếp `is-active` |
> | `not-found` | Unit không tồn tại → **chưa cài gói** | `apt-get install -y qemu-guest-agent` |
>
> Nếu quên `qm set --agent 1` thì trong guest **không có cổng** → udev không kích hoạt → dịch vụ không
> chạy dù gói đã cài.

> **Vì sao cài ở template chứ không cài trên từng VM:** template 9000 là *Golden image* của lớp — VM 130,
> 132 và mọi clone sau này đều thừa hưởng. Cài một lần ở đây thay vì lặp lại trên từng máy chính là
> nguyên tắc golden image. Ngoài lab còn hai cách nữa cho cùng mục đích:
> `virt-customize -a <img> --install qemu-guest-agent` (nướng thẳng vào file ảnh, không cần boot), hoặc
> để cloud-init cài lúc first boot bằng snippet `#cloud-config` có `packages: [qemu-guest-agent]`
> gán qua `qm set --cicustom`.
>
> `cloud-init clean` và reset `machine-id` là hai bước hay bị quên: bỏ qua thì clone kế thừa
> instance-id và machine-id của template — cloud-init có thể không chạy lại, và các dịch vụ dựa trên
> machine-id (journald, DHCP client) sẽ trùng danh tính giữa các VM.

## Bước 2 — Clone: full clone và linked clone

```bash
qm clone 9000 130 --name web-01 --full                # full clone (độc lập)
qm clone 9000 132 --name web-02                        # linked clone (mặc định)

# IP TĨNH (toàn lab không dùng DHCP). VM 130 nằm trên mạng mgmt để jump ping trực tiếp
# -> dùng làm VM chứng minh live migration / HA không đứt kết nối (Bài 05).
qm set 130 --ipconfig0 ip=10.0.10.30/24,gw=10.0.10.1
qm set 132 --ipconfig0 ip=10.0.10.31/24,gw=10.0.10.1
qm list | grep -E '9000|130|132'
```

Xác minh khác biệt:

```bash
qm config 130 | grep scsi0        # full: disk riêng, không tham chiếu base
qm config 132 | grep scsi0        # linked: base là disk của template 9000
```

> Full clone tạo lâu hơn, tốn dung lượng đầy đủ, xóa template vẫn chạy. Linked clone tạo tức thì, tiết kiệm, nhưng **phụ thuộc template 9000** — không được xóa/sửa template.

## Bước 3 — Khởi động VM và xác minh Guest Agent

Khởi động full clone (VM 130) và chờ Cloud-Init cấp phát:

```bash
qm start 130
sleep 45
```

Agent đã được cài sẵn trong template ở Bước 1 nên clone thừa hưởng — xác minh cả hai phía đã thông:

```bash
qm agent 130 ping && echo "AGENT OK"
qm agent 130 network-get-interfaces | grep -o '"ip-address":"[0-9.]*"' | head   # phải thấy 10.0.10.30

# Guest có đường ra Internet qua NAT trên jump (không cần IP ngoài):
qm guest exec 130 -- ping -c1 -W2 10.0.10.1 | grep -q '"exitcode" : 0' && echo "GW OK"
qm guest exec 130 --timeout 120 -- apt-get update | grep -q '"exitcode" : 0' && echo "APT OK"
```

> `qm guest exec` trả JSON chứa `"exitcode"` của lệnh **trong guest**; bản thân `qm` hầu như luôn
> exit 0 kể cả khi lệnh trong guest fail hoặc hết giờ. Vì vậy phải đọc `"exitcode"` — viết
> `qm guest exec ... && echo OK` là tự lừa mình.

Kỳ vọng: `AGENT OK`; `network-get-interfaces` trả về IP guest (chứng tỏ agent đang chạy); `GW OK` và `APT OK`.

> Kiểm chứng golden image có tác dụng thật: `qm start 132 && sleep 45 && qm agent 132 ping && echo "AGENT OK"` — VM 132 là
> **linked clone**, chưa ai động vào nó, nhưng agent vẫn chạy vì nó thừa hưởng từ template. Đó chính
> là lợi ích của việc cài ở Bước 1 thay vì cài trên từng máy.

> Nếu `qm agent 130 ping` vẫn fail sau khi đã cài: kiểm `qm config 130 | grep agent` (phải có
> `agent: 1`) và trong guest `systemctl status qemu-guest-agent`. Thiếu một trong hai phía là hỏng.

## Bước 4 — Snapshot VM trên Ceph

```bash
qm snapshot 130 pre-change --description "trước khi đổi cấu hình"
qm listsnapshot 130
```

Kỳ vọng: snapshot `pre-change` xuất hiện (Ceph RBD hỗ trợ snapshot).

> Thử trên storage KHÔNG hỗ trợ (vd iscsi-lvm từ challenge 02) sẽ bị từ chối — chứng minh "snapshot là thuộc tính của storage". Nhắc: snapshot ≠ backup (Bài 05).

## Bước 5 — Tạo LXC unprivileged + mountpoint

Tải CT template về `nfs-store` (content vztmpl):

```bash
pveam update
# Lấy tên template MỚI NHẤT động (không hard-code phiên bản — repo sẽ xoay vòng):
TMPL=$(pveam available --section system \
       | awk '/debian-.*standard.*_amd64\.tar\./{print $2}' | sort -V | tail -1)
[ -n "$TMPL" ] || { echo "không tìm thấy template debian standard amd64"; exit 1; }
echo "Template: $TMPL"
pveam download nfs-store "$TMPL"
```

Tạo CT 200 (unprivileged) với rootfs trên Ceph và một mountpoint dữ liệu:

```bash
# Lấy đường dẫn volume template vừa tải (khớp bản debian standard mới nhất)
CTTPL=$(pvesm list nfs-store --content vztmpl | awk '/debian-.*standard.*_amd64\.tar\./{print $1}' | sort -V | tail -1)
pct create 200 "$CTTPL" \
  --hostname ct-app --cores 1 --memory 1024 --swap 512 \
  --rootfs vmpool:8 --unprivileged 1 --features nesting=1 \
  --net0 name=eth0,bridge=vmbr0,ip=10.0.10.20/24,gw=10.0.10.1
pct set 200 --nameserver 10.0.10.1 --searchdomain lab.local
pct set 200 --mp0 vmpool:4,mp=/data                   # volume mount 4G tại /data
pct start 200
```

Xác minh:

```bash
pct config 200 | grep -E 'unprivileged|rootfs|mp0'
pct exec 200 -- df -h /data                            # /data mount được
pct exec 200 -- id                                     # chạy được lệnh trong CT
pct exec 200 -- ping -c1 -W2 10.0.10.1                 # gateway (NAT ra Internet)
pct exec 200 -- apt-get update -qq && echo "APT OK"
```

Kỳ vọng: `unprivileged: 1`, `mp0` tại `/data`, CT chạy và `/data` mounted.

## Bước 5b — Cài ứng dụng và cấu hình trong container

> Container chỉ có giá trị khi **chạy được ứng dụng**. Bước này cài một web service, đặt dữ liệu lên
> mountpoint `/data` (không nằm trong rootfs), rồi kiểm chứng từ ngoài CT.

```bash
pct exec 200 -- apt-get install -y nginx
pct exec 200 -- systemctl enable --now nginx

# Nội dung nằm trên MOUNTPOINT /data, không phải rootfs
pct exec 200 -- mkdir -p /data/www
pct exec 200 -- sh -c 'echo "<h1>ct-app on $(hostname)</h1>" > /data/www/index.html'
pct exec 200 -- sh -c "sed -i 's#root /var/www/html;#root /data/www;#' /etc/nginx/sites-available/default"
pct exec 200 -- nginx -t && pct exec 200 -- systemctl reload nginx
```

**Kiểm chứng từ ngoài CT** (từ node hoặc jump host):

```bash
curl -s http://10.0.10.20/            # phải thấy <h1>ct-app on ct-app</h1>
```

**Tối ưu hoá / giới hạn tài nguyên CT — thay đổi nóng, không cần reboot:**

```bash
pct set 200 --cores 2 --memory 2048        # áp ngay, khác hẳn VM (phải tắt/bật)
pct exec 200 -- nproc                      # CT thấy 2 core
pct set 200 --onboot 1                     # tự chạy khi node khởi động
pct config 200 | grep -E 'cores|memory|onboot'
```

> **Note:** CT dùng chung kernel với host nên đổi CPU/RAM **có hiệu lực ngay** — đây là lợi thế
> vận hành lớn so với VM. Đổi lại: không có live migration (xem cuối bài).

> Dữ liệu ứng dụng đặt trên `mp0` (`/data`) thay vì rootfs: rootfs nhỏ và hay bị thay khi rebuild CT,
> còn mountpoint là volume riêng — snapshot/backup và thay đổi kích thước độc lập.

## Bước 6 — Xác nhận cuối (chạy trên pve1)

```bash
echo "== template =="
qm config 9000 | grep -q 'template: 1' && echo "template 9000 OK" || echo "template FAIL"
echo "== clones =="
qm status 130 >/dev/null 2>&1 && qm status 132 >/dev/null 2>&1 && echo "clones OK" || echo "clones FAIL"
echo "== agent VM130 =="
qm agent 130 ping >/dev/null 2>&1 && echo "agent OK" || echo "agent FAIL"
echo "== Golden image hygiene (machine-id phải KHÁC nhau giữa các clone) =="
qm start 132 >/dev/null 2>&1; sleep 30
MID130=$(qm guest exec 130 -- cat /etc/machine-id 2>/dev/null | grep -o '"out-data" : "[^"]*"' | cut -d'"' -f4)
MID132=$(qm guest exec 132 -- cat /etc/machine-id 2>/dev/null | grep -o '"out-data" : "[^"]*"' | cut -d'"' -f4)
if [ -n "$MID130" ] && [ -n "$MID132" ] && [ "$MID130" != "$MID132" ]; then
  echo "machine-id OK (130=${MID130%%\\n*} · 132=${MID132%%\\n*})"
else
  echo "machine-id FAIL — trùng nhau hoặc rỗng: thiếu bước truncate /etc/machine-id ở Bước 1"
fi
qm guest exec 130 -- cloud-init status 2>/dev/null | grep -q 'done' \
  && echo "cloud-init OK (chạy lại trên clone)" || echo "cloud-init FAIL — kiểm 'cloud-init clean' ở Bước 1"
echo "== snapshot VM130 =="
qm listsnapshot 130 | grep -q pre-change && echo "snapshot OK" || echo "snapshot FAIL"
echo "== CT200 =="
pct config 200 | grep -q 'unprivileged: 1' && pct status 200 | grep -q running && echo "CT OK" || echo "CT FAIL"
echo "== storage/cluster nền =="
ceph health | grep -q HEALTH_OK && pvecm status | grep -q 'Quorate: *Yes' && echo "cluster OK" || echo "cluster FAIL"
```

Kỳ vọng: tất cả `OK` — template, clones, agent, **vệ sinh ảnh vàng**, snapshot, CT, và điều kiện nền
(Ceph HEALTH_OK + quorate).

> `machine-id FAIL` nghĩa là hai clone đang dùng chung một danh tính máy. Hậu quả thật: journald trộn
> log giữa các máy, DHCP client xin trùng lease, và mọi thứ định danh theo `/etc/machine-id` (systemd,
> một số agent giám sát) sẽ nhầm máy này với máy kia. Sửa: quay lại Bước 1, `truncate -s 0
> /etc/machine-id` trong template rồi dựng lại clone — **không** sửa tay trên từng clone, vì template
> hỏng thì mọi clone sau vẫn hỏng.
