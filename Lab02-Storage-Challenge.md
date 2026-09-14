# Lab 02 — Challenge: "Tổ Chức Đã Có iSCSI SAN"

> **Kịch bản:** Ban hạ tầng thông báo tổ chức vừa mua một **iSCSI SAN** và muốn tận dụng nó cho cụm Proxmox — không phải mọi thứ đều chạy trên Ceph. SAN đã được cấu hình sẵn, xuất **một LUN qua hai portal** (hai đường mạng độc lập). Nhiệm vụ của bạn: tích hợp LUN này vào cụm **đúng chuẩn multipath** (chịu lỗi một đường), đăng ký làm **shared storage**, và đặt một disk VM lên đó để chứng minh nó dùng được.
>
> Đây là mô hình **"array quản lý redundancy"** — ngược với Ceph (Proxmox quản lý redundancy) bạn vừa dựng ở guided lab. Cả hai cùng tồn tại trong cụm.
>
> **Chấm:** `grade02.sh` từ jump host — kết quả **PASS/FAIL**, PASS khi **tất cả** tiêu chí đạt (kể cả điều kiện nền: Ceph/NFS của guided lab phải còn nguyên vẹn). Chỉ chấm **trạng thái cuối**, không chấm cách làm; 
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
| VM demo | VMID **220** |

## Acceptance criteria

Trên cụm của bạn (grader kiểm chính trên `pve1`):

1. **Điều kiện nền còn nguyên:** cluster quorate; `ceph health` = `HEALTH_OK`; `vmpool` và `nfs-store` vẫn `active`. (Challenge này *thêm vào*, không được phá storage guided.)
2. **iSCSI:** `pve1` đăng nhập LUN qua **cả hai portal** — `iscsiadm -m session` thấy ≥ 2 phiên tới target.
3. **Multipath:** LUN hiện diện dưới `/dev/mapper/` với **≥ 2 path `active ready running`** (`multipath -ll`).
4. **VG trên multipath:** Volume Group `vg_iscsi` nằm trên thiết bị `/dev/mapper/…` (KHÔNG phải trên `/dev/sdX` của một path đơn).
5. **Shared LVM storage:** storage `iscsi-lvm` kiểu LVM, **shared**, đặt trên `vg_iscsi`, trạng thái `active`.
6. **VM disk:** VM **220** tồn tại và có disk nằm trên `iscsi-lvm` (LV `vm-220-disk-*` đã cấp phát trong `vg_iscsi`).

## Cách tự kiểm

Chạy script sau từ jump host:

```bash
cd ~/graders && ./grade02.sh
```
