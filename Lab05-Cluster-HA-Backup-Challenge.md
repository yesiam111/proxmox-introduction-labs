# Lab 05 — Challenge: "Mất Node Lúc Nửa Đêm + Xóa Nhầm VM"

> **Kịch bản:** 2 giờ sáng, `pve3` **mất nguồn đột ngột** — mainboard hỏng, hãng hẹn sáng mai mới thay. VM 130 (app có HA) đang chạy trên đó. Monitoring báo VM 130 **vẫn chưa lên lại** dù cụm còn 2 node khỏe. Cùng đêm, một thao tác sai đã **xóa mất VM 520**. Bạn được gọi dậy.
>
> Việc của bạn: (1) tìm vì sao HA **không** tự hồi phục VM 130 và sửa; (2) **khôi phục VM 520** từ backup — lưu ý: tối qua có người "sửa" VM 520 trước khi backup đêm chạy.
>
> Bài học cốt lõi (Bài 05 slide 24): HA + Ceph **không** cứu "xóa nhầm" — chỉ **backup** (bản tách theo thời gian) mới cứu. Và "bản mới nhất" chưa chắc là "bản đúng".
>
> **Chấm:** `grade05.sh` từ jump host. Kết quả **PASS/FAIL**, PASS khi **tất cả** tiêu chí đạt.

## Luật chơi (vi phạm = sự cố thứ hai)

- **KHÔNG bật lại `pve3`** (kể cả từ Proxmox lớp ngoài). Phần cứng hỏng — bạn phải vận hành cụm **2 node** cho tới sáng.
- **KHÔNG ép quorum** (`pvecm expected …`). Cụm 2/3 vẫn quorate; hạ expected votes là mở cửa cho **split-brain**.
- **KHÔNG xóa HA resource `vm:130`** để start tay — sửa đúng nguyên nhân, để HA làm việc của nó.

## Tên tài nguyên & mục tiêu

| Mục | Giá trị |
|-----|---------|
| Node chết (giữ nguyên trạng thái tắt) | `pve3` |
| VM HA phải hồi phục | VMID **130** — chạy trên node khỏe, HA `started` |
| VM bị xóa, phải restore | VMID **520** — disk `scsi0` trên `vmpool`, **đúng 2G** như trước sự cố |
| Nguồn restore | PBS namespace của bạn (storage `pbs`) |
| RTO target (VM 520) | **≤ 15 phút** (tự bấm giờ; grader chấm trạng thái cuối) |

## Acceptance criteria

1. **Cụm degraded đúng cách:** quorate với 2/3 node; expected votes vẫn là 3; Ceph còn phục vụ I/O (HEALTH_WARN do mất 1 host là **bình thường**, nhưng không được có PG inactive).
2. **HA hồi phục:** VM 130 **running trên node khỏe**, vẫn là HA resource `started`.
3. **PBS namespace:** storage `pbs` cấu hình với **namespace riêng** của bạn.
4. **Restore đúng bản:** VM 520 được `qmrestore` từ PBS, `scsi0` trên `vmpool` với `size=2G`.

## Tra cứu nhanh (cú pháp — KHÔNG phải trình tự lời giải)

Liệt kê theo *tầng*, không theo thứ tự phải chạy. Mọi lệnh đều có `--help` / `man`.

**Cluster & quorum**

```bash
pvecm status ; pvecm nodes           # quorate chưa, node nào online
```

> Mất quorum thì `/etc/pve` chuyển read-only: `qm`, `pvesm`, `ha-manager` đều đứng.
> Cụm 3 node chịu được mất **1** node — trong lúc chờ hồi phục, **đừng đụng vào node thứ hai**.

**HA**

```bash
ha-manager status                    # trạng thái từng resource + node nào là master
ha-manager config                    # cấu hình resource đang khai báo
ha-manager add vm:<vmid> --state started
ha-manager crm-command migrate vm:<vmid> <node>   # resource HA phải đi đường này, KHÔNG dùng qm migrate
ha-manager rules list                             # rule HA (node-affinity / resource-affinity)
cat /etc/pve/ha/rules.cfg                         # cấu hình thô: nodes, strict, resources
ha-manager rules set node-affinity <id> --nodes "<n1>:<prio>,<n2>:<prio>" --strict <0|1>
journalctl -u pve-ha-crm --since "-1h" --no-pager # quyết định của CRM: fence, recovery, vì sao không start
journalctl -u pve-ha-crm -o short-iso | grep -i fenc   # dòng log liên quan fence, có giờ
```

> Trạng thái cần đọc hiểu trong `ha-manager status`: `fence` (đang chờ chắc chắn node chết),
> `recovery` (node đã bị fence, đang tìm node mới — **kẹt ở đây** nghĩa là không có node nào hợp lệ).
> Chỉ **CRM master** ghi log quyết định fence — xem `ha-manager status` để biết master là node nào.
> `strict 1` trong node-affinity: resource **chỉ** được chạy trên các node liệt kê, kể cả khi tất cả đã chết.

> Hồi phục HA không tức thì: CRM phải chờ **agent lock** của node chết hết hạn (~2 phút — watchdog
> bảo đảm node đó đã tự reset/chết) rồi mới coi là **fenced** và khởi động resource ở node khác.
> Chưa qua 2–3 phút mà VM chưa lên là bình thường; **quá** thời gian đó mới là có vấn đề.

**Storage & PBS**

```bash
pvesm status                         # storage nào active
cat /etc/pve/storage.cfg             # xem cấu hình thô, gồm dòng `namespace`
pvesm add pbs <id> --server <ip> --datastore <store> \
        --username <user@pbs> --password --fingerprint <fp> --namespace <ns>
pvesm list <storage-id>              # liệt kê volume/backup trong một storage
```

**Backup / restore**

```bash
vzdump <vmid> --storage pbs --mode snapshot       # tạo backup
pvesm list pbs --vmid <vmid>                      # mọi bản backup: volid (có thời điểm) + SIZE
pvesh get /nodes/<node>/storage/pbs/content --vmid <vmid> --output-format json-pretty   # + notes, ctime
qmrestore <volid> <vmid> --storage vmpool         # restore VM từ backup
qm start <vmid> ; qm status <vmid>
```

Phía PBS (nếu cần kiểm tra trực tiếp trên server backup):

```bash
proxmox-backup-client snapshot list --repository <user@pbs>@<ip>:<store> --ns <namespace>
proxmox-backup-manager verify <store> --ns <namespace>
```

**Điều kiện nền**

```bash
ceph health detail ; ceph -s         # mất 1 host: HEALTH_WARN (degraded/undersized) là bình thường;
                                     # PG "inactive"/PG_AVAILABILITY mới là I/O đứng
```

## Cách tự kiểm

```bash
cd ~/graders && ./grade05.sh      # grader có poll — có thể chạy vài phút
```
