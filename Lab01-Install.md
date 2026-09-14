# Lab 01 (Guided) — Cài Đặt Proxmox VE và Chuẩn Bị Node

> Xây trạng thái nền cho toàn khóa: 3 node Proxmox VE cài sạch, network/hostname/repo/NTP đúng chuẩn, disk sẵn sàng cho Bài 02.
> **Chấm điểm:** không chấm điểm — nhưng cuối lab có lệnh xác nhận bắt buộc chạy sạch trước khi instructor snapshot **checkpoint 1**.

## Sơ đồ môi trường (per student)

| VM | vCPU | RAM | Disk | Vai trò |
|----|------|-----|------|---------|
| pve1, pve2, pve3 | 4 | 12 GB | 1× 32 GB (OS) + 3× 25 GB (dữ liệu — để trống) | Node Proxmox VE |

Quy hoạch network — **4 NIC / 3 IP** mỗi node (tên NIC có thể khác, xác định bằng `ip -br a`):

| NIC | Mạng | Subnet | pve1 | pve2 | pve3 | Dùng cho |
|-----|------|--------|------|------|------|----------|
| 1 | Management | 10.0.10.0/24 | .11 | .12 | .13 | Web UI, SSH; Bài 04 vào bond0 |
| 2 | Bond slave (VM/tenant) | — (VLAN trên vmbr0) | — | — | — | Bài 04 (bond active-backup) |
| 3 | Ceph (public+cluster gộp) | 10.0.30.0/24 | .11 | .12 | .13 | Bài 02 (MTU 9000) |
| 4 | Corosync | 10.0.20.0/24 | .11 | .12 | .13 | Bài 02 (cluster) / Bài 05 |

Gateway/DNS/NTP: `10.0.10.1` (jump host). Domain: `lab.local`. Bài 01 chỉ cấu hình NIC 1 (mgmt → vmbr0); NIC 3/4 cấu hình ở Bài 02, bond ở Bài 04.

---

## Bước 1 — Cài Proxmox VE (làm trên cả 3 node)

Mở console của node từ outer hypervisor (đã mount sẵn ISO Proxmox VE):

1. Boot ISO → **Install Proxmox VE (Graphical)** → đồng ý EULA.
2. **Target disk:** chọn disk **32 GB** (disk OS). Ba disk 25 GB KHÔNG đụng tới. Options để mặc định (ext4/LVM).
3. Country/Timezone: `Vietnam` / `Asia/Ho_Chi_Minh`. Mật khẩu root + email theo quy ước lớp.
4. **Network:**
   - NIC: chọn NIC **thứ nhất** (management).
   - FQDN: `pveN.lab.local` (N = 1/2/3 theo node đang cài).
   - IP: `10.0.10.1N/24` · Gateway: `10.0.10.1` · DNS: `10.0.10.1`.
5. Xác nhận → cài → reboot. Lặp lại cho đủ 3 node.

## Bước 2 — Xác minh network và hostname (mỗi node)

SSH vào từng node: `ssh root@10.0.10.11` (12, 13):

```bash
ip -br a                      # NIC1 mang IP mgmt, vmbr0 UP
hostname --ip-address         # PHẢI ra 10.0.10.1N — KHÔNG phải 127.0.1.1
cat /etc/hosts
```

Thêm đủ 3 node vào `/etc/hosts` của **mỗi** node (installer chỉ tạo dòng của chính nó):

```bash
cat >> /etc/hosts <<'EOF'
10.0.10.11 pve1.lab.local pve1
10.0.10.12 pve2.lab.local pve2
10.0.10.13 pve3.lab.local pve3
EOF
sed -i '/^127.0.1.1/d' /etc/hosts     # xóa dòng 127.0.1.1 nếu có
```

Kiểm tra chéo — từ pve1:

```bash
ping -c2 pve2 && ping -c2 pve3        # phải chạy bằng HOSTNAME, không chỉ IP
```

## Bước 3 — Repository và cập nhật (mỗi node)

Mặc định enterprise repo được bật → `apt update` trả **401 Unauthorized**. Chuyển sang no-subscription (PVE 9 dùng định dạng deb822):

```bash
rm -f /etc/apt/sources.list.d/pve-enterprise.sources
rm -f /etc/apt/sources.list.d/ceph.sources      # ISO PVE 9 con repo ceph enterprise -> cung tra 401
cat > /etc/apt/sources.list.d/proxmox.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
apt update                            # phải sạch — không 401
apt -y dist-upgrade                   # dist-upgrade, KHÔNG dùng upgrade
```

Kiểm tra: `pveversion` — cả 3 node phải cùng phiên bản.

## Bước 4 — NTP với chrony (mỗi node)

```bash
sed -i 's/^pool .*/server 10.0.10.1 iburst/' /etc/chrony/chrony.conf
systemctl enable --now chrony
chronyc tracking                      # System time offset phải rất nhỏ (< 0.1s)
chronyc sources -v                    # nguồn 10.0.10.1, trạng thái ^*
```

> Vì sao quan trọng: Ceph cảnh báo khi MON lệch > **0.05 giây**. Cả 3 node phải cùng nguồn.

## Bước 5 — Disk inventory (mỗi node)

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,ROTA,MOUNTPOINT
ls -l /dev/disk/by-id/ | grep -v part
```

Xác nhận: disk 32 GB mang OS (có partition, mounted); **3 disk 25 GB hoàn toàn trống** — không partition, không filesystem, không LVM. Ghi lại tên `by-id` của 3 disk trống vào file ghi chú của bạn — Bài 02 dùng.

> Không bao giờ tham chiếu disk bằng `/dev/sdX` trong tài liệu vận hành — tên này có thể đổi sau reboot.

## Bước 5b — RAID: mức RAID nào cho disk nào (thực hành trên pve1)

> Lab này có **1 disk OS + 3 disk trống**, nên disk OS cài
> single-disk; ba disk trống là chỗ **thực hành thật** các mức RAID của ZFS — làm xong **huỷ pool**
> vì Bài 02 cần ba disk đó còn trống cho Ceph OSD.

**Chỉ làm trên pve1.** Lấy tên `by-id` của 3 disk trống từ Bước 5:

```bash
D1=/dev/disk/by-id/<disk1>; D2=/dev/disk/by-id/<disk2>; D3=/dev/disk/by-id/<disk3>

# a) MIRROR (RAID1) — 2 disk, chịu mất 1 disk, dung lượng = 1 disk
zpool create -o ashift=12 labtest mirror "$D1" "$D2"
zpool status labtest
zpool list labtest        # SIZE ~24.5G — vdev mirror = kích thước MỘT disk
zfs  list labtest         # AVAIL ~24G  — trùng khớp, không có gì bất ngờ
zpool destroy labtest

# b) RAIDZ1 (~RAID5) — 3 disk, chịu mất 1 disk, dung lượng = 2 disk
zpool create -o ashift=12 labtest raidz1 "$D1" "$D2" "$D3"
zpool status labtest
zpool list labtest        # SIZE ~74.5G — RAW (3×25G, ĐÃ TÍNH CẢ PARITY)
zfs  list labtest         # AVAIL ~48G  — dung lượng DÙNG ĐƯỢC thật sự
```

> ⚠️ **`zpool list` đọc khác nhau giữa mirror và raidz — đây là chỗ nhầm kinh điển.**
>
> | | `zpool list` SIZE | Dung lượng dùng được |
> |---|---|---|
> | mirror 2×25G | **24.5G** (= 1 disk) | ~24G — khớp |
> | raidz1 3×25G | **74.5G** (= raw cả parity) | ~48G — **KHÔNG khớp** |
>
> Với raidz, luôn xem `zfs list` (hoặc `zpool list -v`) để biết còn bao nhiêu chỗ thật.
> Thấy 74.5G rồi tưởng chứa được 74G là cách nhanh nhất để làm đầy pool ngoài ý muốn.

**Thử hỏng một disk (bằng chứng RAID hoạt động):**

```bash
zpool offline labtest "$D2"
zpool status labtest        # state: DEGRADED — pool VẪN đọc/ghi được
zpool online  labtest "$D2"
zpool status labtest        # resilver xong -> ONLINE
```

**Huỷ pool — BẮT BUỘC trước Bài 02:**

```bash
zpool destroy labtest
wipefs -a "$D1" "$D2" "$D3"
lsblk -o NAME,SIZE,TYPE,FSTYPE   # 3 disk phải trống trở lại
```

| Mức | Tối thiểu | Chịu mất | Dung lượng dùng được | Dùng khi |
|---|---|---|---|---|
| single / RAID0 | 1 / 2 | **0** | 100% | lab, dữ liệu bỏ đi được |
| mirror (RAID1) | 2 | 1 | 50% | disk OS, pool cần IOPS ghi cao |
| RAIDZ1 | 3 | 1 | ~67% | dung lượng lớn, ghi tuần tự |
| RAIDZ2 | 4 | 2 | ~50% | disk lớn (rebuild lâu → cần 2 lớp dự phòng) |

> ⚠️ **Không chồng RAID cứng dưới ZFS.** ZFS cần thấy từng disk thật để tự sửa lỗi; controller RAID
> giấu disk đi và vô hiệu hoá cơ chế đó. Có RAID controller thì đặt ở chế độ HBA/IT mode.

## Bước 6 — Xác nhận cuối (chạy trên MỖI node)

```bash
echo "== $(hostname) =="
grep -cE 'vmx|svm' /proc/cpuinfo
hostname --ip-address
apt-get update -qq >/dev/null 2>&1 && echo "repo OK" || echo "repo FAIL"
chronyc tracking | grep "System time"
ping -c1 -W1 pve1 >/dev/null && ping -c1 -W1 pve2 >/dev/null && ping -c1 -W1 pve3 >/dev/null && echo "hosts OK" || echo "hosts FAIL"
lsblk -dn -o TYPE | grep -cx disk
```

Kỳ vọng: số > 0 (VT-x) · IP thật 10.0.10.1N · `repo OK` · offset nhỏ · `hosts OK` · 4 disk.

**Báo instructor khi cả 3 node sạch** → instructor snapshot **checkpoint 1** → nhận challenge.
