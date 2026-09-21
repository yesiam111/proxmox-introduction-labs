# Lab 03 — Challenge: "Appliance Của Nhà Cung Cấp"

> **Kịch bản:** Một nhà cung cấp giao cho bạn một **appliance dạng OVA** (`appliance.ova`) để triển khai lên cụm Proxmox. Bạn import nó vào — nhưng bấm **Start** thì màn hình đen, không boot; và kể cả khi chạy được thì Proxmox cũng không "nói chuyện" được với nó. Nhiệm vụ: đưa appliance về trạng thái **boot được, chạy trên VirtIO SCSI, và quản lý được qua QEMU Guest Agent** — để hệ thống giám sát đọc được token cấu hình bên trong.

>
> **Chấm:** `grade03.sh` từ jump host — kết quả **PASS/FAIL**, PASS khi **tất cả** tiêu chí đạt (kể cả điều kiện nền: Ceph/cluster của guided còn nguyên).

## Vị trí appliance (đã đặt sẵn)

Nhìn thấy trên mọi node qua NFS storage của bài trước:

```
/mnt/pve/nfs-store/import/appliance.ova
    (kèm appliance.ovf + appliance-disk.vmdk + appliance-disk.qcow2 nếu cần)
```

## Tên tài nguyên BẮT BUỘC

| Tài nguyên | Giá trị bắt buộc |
|-----------|------------------|
| VMID | **320** |
| Storage đặt đĩa | `vmpool` (Ceph) |

## Acceptance criteria

Grader kiểm chính trên `pve1`:

1. **Điều kiện nền còn nguyên:** cluster quorate; `ceph health` = `HEALTH_OK`.
2. **VM 320 tồn tại** (đã import).
3. **VM 320 đang chạy** — `qm status 320` = `running` (đã boot thành công).
4. **Đĩa trên VirtIO SCSI:** `qm config 320` có `scsihw: virtio-scsi-single`, một dòng `scsi0:`, và `boot: order=scsi0`.
5. **Guest Agent bật ở VM:** `qm config 320` có `agent: 1`.
6. **Guest Agent phản hồi:** `qm agent 320 ping` thành công.
7. **Đọc được token trong guest qua agent:** `qm guest exec 320 -- cat /opt/lab/token.txt` trả về đúng token.

## Tra cứu nhanh

**Import**

```bash
qm importovf <vmid> <duong-dan>.ovf <storage>     # đọc OVF, tạo VM, GẮN đĩa vào bus OVF khai
qm disk import <vmid> <file.qcow2|.vmdk> <storage>  # chỉ nạp đĩa -> về unusedN (đường dự phòng)
```

> `importovf` chỉ lấy `name` / `memory` / `cores` từ OVF. NIC, serial port… đều **không** được tạo.

**Xem và sửa cấu hình VM**

```bash
qm config <vmid>                                  # LUÔN xem trước khi sửa
qm set <vmid> --scsihw virtio-scsi-single
qm set <vmid> --scsi0 <storage>:<volume>,discard=on,ssd=1
qm set <vmid> --delete <bus>                      # gỡ đĩa khỏi bus cũ -> volume về unusedN
qm set <vmid> --boot order=scsi0
qm set <vmid> --agent 1                           # bật agent Ở MỨC VM (chưa phải trong guest)
qm set <vmid> --serial0 socket                    # cần có thì `qm terminal` mới chạy
qm set <vmid> --net0 virtio,bridge=vmbr0
```

> Gắn cùng một volume lên hai bus (vd còn `sata0` lại thêm `scsi0`) là lỗi — VM sẽ không start.
> Gỡ bus cũ trước, rồi xoá tham chiếu `unusedN` còn lại.

**Chạy và quan sát**

```bash
qm start <vmid> ; qm status <vmid> ; qm stop <vmid>
qm terminal <vmid>                                # cần serial0; thoát bằng Ctrl+O
qm agent <vmid> ping                              # agent trong guest có trả lời không
qm guest exec <vmid> -- <lenh>                    # chạy lệnh trong guest qua agent
```

> `qm guest exec` trả về JSON chứa `"exitcode"` của lệnh trong guest, còn bản thân `qm` hầu như
> luôn exit 0 — muốn biết lệnh trong guest thành công hay không thì phải đọc `"exitcode"`.

**Bên trong guest** (nếu agent chưa phản hồi)

```bash
apt-get install -y qemu-guest-agent
systemctl is-active qemu-guest-agent     # 'static'/không enable được là ĐÚNG: unit kích hoạt theo
systemctl start qemu-guest-agent         # thiết bị (udev bắt cổng virtio khi VM có --agent 1)
```

## Cách tự kiểm

```bash
cd ~/graders && ./grade03.sh
```
