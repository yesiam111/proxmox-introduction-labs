# Lab 02 — Challenge: "Tổ Chức Đã Có iSCSI SAN"

> **Kịch bản:** Ban hạ tầng thông báo tổ chức vừa mua một **iSCSI SAN** và muốn tận dụng nó cho cụm Proxmox — không phải mọi thứ đều chạy trên Ceph. SAN đã được cấu hình sẵn, xuất **một LUN qua hai portal** (hai đường mạng độc lập). Nhiệm vụ của bạn: tích hợp LUN này vào cụm **đúng chuẩn multipath** (chịu lỗi một đường), đăng ký làm **shared storage**, và **cấp phát một volume** trên đó để chứng minh nó dùng được. (Chưa tạo VM — thao tác VM/CT là nội dung Bài 03.)
>
> Đây là mô hình **"array quản lý redundancy"** — ngược với Ceph (Proxmox quản lý redundancy) bạn vừa dựng ở guided lab. Cả hai cùng tồn tại trong cụm.
>
> **Chấm:** `grade02.sh` từ jump host — kết quả **PASS/FAIL**, PASS khi **tất cả** tiêu chí đạt (kể cả điều kiện nền: Ceph/NFS của guided lab phải còn nguyên vẹn). Chỉ chấm **trạng thái cuối**, không chấm cách làm; không điểm số, không hình phạt — FAIL thì tiếp tục sửa. Được tra tài liệu thoải mái.
> Challenge này **khác** guided lab: guided làm Ceph, challenge làm iSCSI + multipath — chép lại lệnh của guided lab không giúp gì ở đây.
## Thông tin SAN (đã dựng sẵn)

| Mục | Giá trị |
|-----|---------|
| Target IQN | `iqn.2026-07.local.lab:san.lun0` |
| Portal 1 (path 1) | `10.0.10.1:3260` (mạng management) |
| Portal 2 (path 2) | `10.0.30.1:3260` (mạng storage — path độc lập) |
| LUN | 1 × 5 GB |
| Xác thực | không CHAP (demo mode) |

## Tên tài nguyên BẮT BUỘC (grader kiểm đúng tên)

Dùng chính xác các tên sau — grader dựa vào chúng (namespace cố định cho challenge 02, tránh va chạm với tài nguyên module khác):

| Tài nguyên | Tên bắt buộc |
|-----------|--------------|
| LVM Volume Group | `vg_iscsi` |
| Proxmox storage (LVM shared) | `iscsi-lvm` |
| Owner ID của volume | **220** (chỉ là *id sở hữu* volume — challenge này chưa tạo VM) |

## Acceptance criteria

Trên cụm của bạn (grader kiểm chính trên `pve1`):

1. **Điều kiện nền còn nguyên:** cluster quorate; `ceph health` = `HEALTH_OK`; `vmpool` và `nfs-store` vẫn `active`. (Challenge này *thêm vào*, không được phá storage guided.)
2. **iSCSI:** `pve1` đăng nhập LUN qua **cả hai portal** — `iscsiadm -m session` thấy ≥ 2 phiên tới target.
3. **Multipath:** LUN hiện diện dưới `/dev/mapper/` với **≥ 2 path `active ready running`** (`multipath -ll`).
4. **VG trên multipath:** Volume Group `vg_iscsi` nằm trên thiết bị `/dev/mapper/…` (KHÔNG phải trên `/dev/sdX` của một path đơn).
5. **Shared LVM storage:** storage `iscsi-lvm` kiểu LVM, **shared**, đặt trên `vg_iscsi`, trạng thái `active`.
6. **Cấp phát được volume:** trên `iscsi-lvm` có volume `vm-220-disk-0` kích thước 4 GiB — tức LV `vm-220-disk-0` tồn tại trong `vg_iscsi`. (Đây là bài *storage*: chứng minh storage cấp phát được volume. Gắn volume vào VM là việc của Bài 03.)

## Tra cứu nhanh

Mọi lệnh đều có `--help` / `man`; bản đầy đủ: `man iscsiadm`, `man multipath.conf`, `man pvesm`, `pvesm help alloc`.

**Gói cần có trên node** (PVE không cài sẵn `multipath-tools`)

```bash
apt-get update && apt-get install -y open-iscsi multipath-tools
```

**Tầng iSCSI initiator** — `iscsiadm`

```bash
iscsiadm -m discovery -t sendtargets -p <portal-ip>     # hỏi target có những gì (trả về CẢ các portal)
iscsiadm -m node -T <iqn> -p <portal-ip>:3260 --login   # đăng nhập MỘT portal
iscsiadm -m session                                     # đang có mấy phiên, tới portal nào
iscsiadm -m node -T <iqn> -o update \
         -n node.startup -v automatic                   # tự login lại sau reboot
iscsiadm -m node -T <iqn> -p <portal> --logout          # gỡ khi làm lại
```

**Tầng multipath** — gộp nhiều `/dev/sdX` của cùng một LUN thành một `/dev/mapper/mpathX`

```bash
systemctl enable --now multipathd
multipath -ll                 # xem map + trạng thái từng path ('active ready running')
multipath -r                  # nạp lại cấu hình / gộp lại path
lsblk                         # đối chiếu: mỗi path là một /dev/sdX
```

`/etc/multipath.conf` — tối thiểu cho lab này:

```
defaults {
    user_friendly_names yes
    find_multipaths     yes      # chỉ gộp thiết bị có >= 2 path (tránh ôm nhầm disk OSD)
}
```

**Tầng LVM** — đặt PV/VG **trên thiết bị multipath**, không trên `/dev/sdX` đơn lẻ

```bash
pvcreate /dev/mapper/<mpath>
vgcreate <ten-vg> /dev/mapper/<mpath>
pvs ; vgs ; lvs <ten-vg>       # kiểm PV nằm trên /dev/mapper/..., liệt kê LV
```

**Tầng Proxmox storage** — `pvesm`

```bash
pvesm add lvm <storage-id> --vgname <ten-vg> \
        --shared 1 --content images            # đăng ký LVM trên VG CÓ SẴN
pvesm status                                   # type / status / dung lượng từng storage
pvesm list <storage-id>                        # liệt kê volume trong storage
pvesm alloc <storage-id> <vmid> <filename> <size>   # cấp phát volume, vd: ... 220 vm-220-disk-0 4G
pvesm free <storage-id>:<volume>               # xoá volume khi cần làm lại
```

> `pvesm alloc` cấp phát ở **tầng storage** — không cần VM nào tồn tại. Tham số `<vmid>` chỉ ghi *ai sở hữu*
> volume, dùng để đặt tên và để PVE biết dọn theo VM nào sau này. Thao tác VM/CT là nội dung **Bài 03**.

**Lệnh kiểm tra điều kiện nền** (đừng để challenge này làm hỏng guided lab)

```bash
pvecm status | grep Quorate ; ceph health ; pvesm status
```

## Cách tự kiểm

Chạy script sau từ jump host:

```bash
cd ~/graders && ./grade02.sh
```
