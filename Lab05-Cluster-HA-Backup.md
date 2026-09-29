# Lab 05 (Guided) — HA, Fencing và Backup với PBS

> Bật High Availability cho VM (trên shared storage Ceph), xác nhận fencing, và kết nối Proxmox Backup Server (namespace + user riêng) để backup + verify + test restore. Đặt nền cho challenge failover/restore.
> **Chấm điểm:** không chấm điểm — cuối lab có khối lệnh xác nhận bắt buộc chạy sạch trước khi instructor snapshot **checkpoint 5**.

## Prerequisites

- Đã hoàn tất **guided + challenge Bài 04** (checkpoint 4): cluster quorate, Ceph `HEALTH_OK`, mạng đúng chuẩn, user RBAC + firewall.
- **PBS chung** (instructor dựng sẵn — spec §5): `pbs0` tại `10.0.10.5`, datastore `store`. Instructor cấp cho bạn: **namespace riêng**, **PBS user/token**, và **fingerprint**.
- VM 130 (từ Lab 03/04) đang có, disk trên `vmpool` (Ceph — shared, đủ điều kiện HA).

## Quy ước lab này

| Tài nguyên | Giá trị |
|-----------|---------|
| VM đưa vào HA | VMID **130** |
| VM được backup & bảo vệ | VMID **520** |
| HA Group | `prod` |
| PBS storage (trong PVE) | tên `pbs`, namespace = **của bạn** (instructor cấp) |

---

## Bước 1 — Đưa VM vào HA + Node Affinity Rule

Thêm VM 130 vào HA, rồi tạo node affinity rule ưu tiên pve1 > pve2 > pve3:

```bash
# 1) HA resource (failback 0 = hành vi 'nofailback' cũ)
ha-manager add vm:130 --state started --max_restart 3 --max_relocate 3 --failback 0

# 2) Node affinity rule: ưu tiên cao hơn = số lớn hơn (bỏ số => priority 0)
ha-manager rules add node-affinity prod --resources vm:130 --nodes "pve1:3,pve2:2,pve3:1"

ha-manager status
ha-manager rules list
```

Kỳ vọng: `ha-manager status` hiện `vm:130 (…, started)`; `ha-manager rules list` hiện rule `prod` kiểu `node-affinity`.

> HA chỉ start được VM ở node khác khi disk trên **shared storage** — VM 130 nằm trên `vmpool` (Ceph). Nếu VM còn disk local, HA sẽ không failover được (nhắc Bài 05 slide 14).

## Bước 2 — Xác nhận fencing (watchdog)

```bash
cat /etc/default/pve-ha-manager | grep -i watchdog     # WATCHDOG_MODULE (softdog mặc định)
ha-manager status                                       # LRM mỗi node: active
lsmod | grep -E 'softdog|wdat|iTCO'                     # module watchdog đang nạp
```

Kỳ vọng: watchdog module nạp (softdog trong lab), LRM `active` trên các node. Đây là cơ chế self-fence khi mất quorum (slide 16).

## Bước 3 — Kết nối PBS (namespace + user riêng)

Dùng thông tin instructor cấp — **namespace và user là hai thứ khác nhau**, ví dụ cặp số 01:
namespace `team-01`, user `backup-01@pbs`.

```bash
pvesm add pbs pbs \
  --server 10.0.10.5 --datastore store \
  --namespace team-01 \
  --username backup-01@pbs --password '<PASS>' \
  --fingerprint <FINGERPRINT>
pvesm status | grep pbs                 # 'pbs' phải active
```

> **User PBS luôn có realm: `tên@realm`** (`@pbs` cho user nội bộ của PBS, `@pam` cho user Linux của
> máy PBS). Thiếu realm, PBS từ chối ngay: `error fetching datastores - 400 Bad Request`.
> Phân biệt nhanh: **400** = định dạng sai (thường là thiếu `@pbs`); **401** = sai mật khẩu;
> lỗi về **fingerprint** = chép sai/thiếu chuỗi fingerprint.

Kỳ vọng: storage `pbs` active, trỏ đúng **namespace của bạn**.

> Namespace cách ly logic trên datastore chung: bạn chỉ thấy/ghi backup trong namespace của mình; chunk store chung nên dedup giữa học viên rất cao (slide 27-29). User của bạn chỉ có quyền `DatastoreBackup` trên namespace này — không xóa được của người khác.

## Bước 4 — Tạo VM được bảo vệ (520) + backup + verify

Tạo một VM nhỏ trên Ceph để backup:

```bash
qm create 520 --name protected-app --memory 512 --cores 1 \
  --scsihw virtio-scsi-single --net0 virtio,bridge=vmbr0 --agent 1
qm set 520 --scsi0 vmpool:2                    # disk 2G trên Ceph
```

Backup VM 520 lên PBS (mode snapshot) và verify:

```bash
vzdump 520 --storage pbs --mode snapshot --notes-template 'lab05 protected'
pvesm list pbs | grep 520                      # thấy bản backup vm/520
```

**Verify (bước bắt buộc — do INSTRUCTOR chạy):** ACL của học viên chỉ có `DatastoreBackup` nên
**không** verify được (verify cần quyền cấp datastore). Instructor (tài khoản admin PBS) chạy verify cho
namespace của học viên, rồi báo kết quả:

```bash
# TRÊN pbs0 (admin PBS): verify thủ công cả datastore (bao gồm namespace của mọi học viên)
proxmox-backup-manager verify store --ignore-verified false
# Xem snapshot trong namespace của một học viên:
proxmox-backup-client snapshot list --repository 'admin@pbs@10.0.10.5:store' --ns <NS>
```

> Verify theo **namespace** cụ thể: dùng **Verify Job** trong GUI (Datastore → Verify Jobs, có trường `ns`), hoặc verify từng snapshot ở tab Content. `proxmox-backup-manager verify <store>` là lệnh verify thủ công theo tài liệu PBS 4.x.

Kỳ vọng: `pvesm list pbs` hiện một bản `vm/520/...`; instructor xác nhận verify **OK**. Ghi lại — challenge sẽ xóa VM 520 và bạn restore từ đây.

> "Untested backup is not a backup" (slide 30): verify (instructor) kiểm checksum, nhưng **test restore thật** ở bước sau mới là bằng chứng backup dùng được. Grader challenge kiểm bản backup **tồn tại đúng namespace** (không kiểm verified — vì việc verify thuộc quyền admin, đã làm ở đây).

## Bước 5 — Test restore (bằng chứng backup dùng được)

Restore bản backup **mới nhất** của 520 sang một VMID tạm (599) — **không** restore đè lên 520 đang chạy — rồi xóa VM tạm:

```bash
pvesm list pbs --vmid 520                      # mọi bản backup của 520, cũ → mới
BK=$(pvesm list pbs --vmid 520 | awk '/vm\/520\//{v=$1} END{print v}')   # dòng CUỐI = mới nhất
echo "$BK"                                     # phải khác rỗng, dạng pbs:backup/vm/520/<thời điểm>
qmrestore "$BK" 599 --storage vmpool
qm config 599 | grep -E '^(name|scsi0|net0):'  # disk về vmpool, size=2G; net0 cùng MAC với 520
qm destroy 599 --purge                         # dọn VM test (KHÔNG start 599)
```

Kỳ vọng: `qmrestore` thành công, VM 599 có `scsi0: vmpool:vm-599-disk-0,…,size=2G` → backup của 520 đọc và ghi ra được.

> **Vì sao chọn dòng cuối?** `pvesm list` sắp theo volid, volid chứa thời điểm backup → dòng cuối là bản mới nhất. `{print $1; exit}` (bản cũ) lấy bản **cũ nhất** — khi đã chạy `vzdump` vài lần, bạn sẽ test nhầm bản không phải bản cần restore.
>
> **Vì sao không start 599?** Restore mang theo **nguyên config** của 520, kể cả MAC trên `net0`. Start 599 khi 520 đang chạy = hai VM cùng MAC trên cùng bridge. Nếu cần boot thử, cắt mạng trước: `qm set 599 --net0 "$(qm config 599 | sed -n 's/^net0: //p'),link_down=1"`.
>
> **Giới hạn của bài test này:** VM 520 ở Bước 4 là VM rỗng (chưa có OS), nên ở đây chỉ chứng minh được *dữ liệu đi ra khỏi PBS nguyên vẹn*, chưa chứng minh được *ứng dụng chạy lại*. Trên hệ thống thật, test restore đầy đủ = restore sang VMID/mạng cô lập → boot → kiểm dịch vụ bên trong (ví dụ `qm agent <id> ping`, `qm guest exec <id> -- systemctl is-active <dịch vụ>`) → xóa.

## Bước 5b — Backup TOÀN HỆ THỐNG (cấu hình host & cluster)

> **PBS backup GUEST, không backup HOST.** Mất một node là mất `/etc/pve` trên host đó, cấu hình
> mạng, danh sách storage, user/ACL… Không có thì "restore VM" xong vẫn không có cụm để chạy.


### Cái gì cần backup

| Đường dẫn / lệnh | Chứa gì | Ghi chú |
|---|---|---|
| `/etc/pve/` | cluster.conf, storage.cfg, user.cfg, firewall, **config mọi guest** | pmxcfs — **chỉ đọc được khi còn quorum** |
| `/etc/network/interfaces` | bond, bridge, VLAN, MTU | mỗi node **khác nhau** — lấy từng node |
| `/etc/hosts`, `/etc/resolv.conf`, `/etc/chrony/` | tên, DNS, NTP | nền của cluster |
| `/var/lib/pve-cluster/config.db` | CSDL pmxcfs | ảnh chụp nhất quán |
| `/etc/ceph/` | ceph.conf, keyring | không có keyring = không vào được cluster Ceph |
| `pvesm status`, `pveum user list`, `pvecm status` | ảnh chụp trạng thái để đối chiếu khi dựng lại | dạng văn bản, dễ đọc lúc khẩn cấp |

### Thực hành — chạy trên **từng node**

```bash
D=/var/backup-host/$(hostname)-$(date +%F)
mkdir -p "$D"

tar czf "$D/etc-pve.tar.gz"        /etc/pve 2>/dev/null
tar czf "$D/etc-network.tar.gz"    /etc/network/interfaces /etc/hosts /etc/resolv.conf
tar czf "$D/etc-ceph.tar.gz"       /etc/ceph 2>/dev/null
cp /var/lib/pve-cluster/config.db  "$D/config.db"

# Ảnh chụp trạng thái (đọc được bằng mắt khi dựng lại)
pvecm status      > "$D/pvecm-status.txt"  2>&1
pvesm status      > "$D/storage.txt"       2>&1
pveum user list   > "$D/users.txt"         2>&1
ip -br a          > "$D/ip.txt"            2>&1
ls -lh "$D"
```

**Đẩy ra ngoài node** — bản backup nằm trên chính node sắp chết thì vô nghĩa:

```bash
# Lên NFS dùng chung của lớp (thật thì đưa ra khỏi site)
mkdir -p /mnt/pve/nfs-store/host-config && cp -r "$D" /mnt/pve/nfs-store/host-config/
```

> `vzdump` **không** bao gồm các file này. Nhiều đội chỉ phát hiện khi dựng lại node và không nhớ nổi
> bond/VLAN/MTU đã cấu hình thế nào.

### Kiểm chứng: mở bản backup ra và đọc được

```bash
tar tzf "$D/etc-pve.tar.gz" | head
tar xzf "$D/etc-network.tar.gz" -O etc/network/interfaces | grep -E 'bond|bridge|vlan|mtu'
```

Kỳ vọng: thấy đúng cấu hình bond/VLAN/MTU của node — đây là thứ dùng để dựng lại node.

> **Quy trình dựng lại một node (không thực hiện trong lab — nắm trình tự):**
> cài PVE cùng phiên bản → khôi phục `/etc/network/interfaces` → `pvecm add` vào cụm hiện có
> (config đồng bộ lại từ pmxcfs) → khôi phục `/etc/ceph` nếu là node Ceph → restore guest từ PBS.
> Ghi chú: `pvecm add` **join lại**, không phải "đổ ngược" `/etc/pve` — bản backup `/etc/pve` dùng để
> **đối chiếu**, và để cứu khi mất *cả cụm*.

## Bước 6 — Xác nhận cuối (chạy trên pve1)

```bash
echo "== HA =="
ha-manager status | grep -q 'vm:130' && ha-manager status | grep 'vm:130' | grep -q started && echo "HA OK" || echo "HA FAIL"
echo "== watchdog =="
lsmod | grep -qE 'softdog|iTCO|wdat' && echo "watchdog OK" || echo "watchdog FAIL"
echo "== PBS namespace =="
grep -A6 '^pbs: pbs' /etc/pve/storage.cfg | grep -q 'namespace' && echo "PBS ns OK" || echo "PBS ns FAIL"
echo "== backup 520 =="
pvesm list pbs | grep -q 'vm/520' && echo "backup OK" || echo "backup FAIL"
echo "== VM520 =="
qm config 520 >/dev/null 2>&1 && echo "VM520 OK" || echo "VM520 FAIL"
echo "== backup cấu hình host =="
ls /var/backup-host/$(hostname)-*/etc-pve.tar.gz >/dev/null 2>&1 && echo "host-config OK" || echo "host-config FAIL"
echo "== nền =="
pvecm status | grep -q 'Quorate: *Yes' && ceph health | grep -q HEALTH_OK && echo "cluster OK" || echo "cluster FAIL"
```

Kỳ vọng: tất cả `OK`.

**Báo instructor khi mọi dòng OK** → instructor snapshot **checkpoint 5** → nhận challenge.

> Sau checkpoint 5, instructor chạy `setup05.sh`: một node **mất nguồn và không quay lại**, VM 520 bị **xóa**, và có thêm vài "thay đổi của người khác" bạn phải tự tìm ra. Challenge: đưa VM 130 chạy lại dưới HA, **restore đúng bản** của VM 520 từ PBS namespace của bạn trong RTO target. Backup vẫn còn trong PBS dù VM đã bị xóa — đó là điểm mấu chốt (HA/Ceph không cứu 'xóa nhầm', chỉ backup cứu được).
