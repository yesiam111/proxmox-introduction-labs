# Lab 06 — Capstone: Sự cố Production

> Bài cuối khóa là **một ca trực sự cố**, không phải bài dựng. Cụm của bạn đang chạy một ứng dụng production (**VM 900**) được dựng sẵn đúng chuẩn — rồi nhiều thứ đã hỏng cùng lúc. Bạn tiếp nhận ticket, chẩn đoán, sửa cho tới khi hệ thống **khỏe lại hoàn toàn**.
>
> **Thời gian:** 2.5–3 giờ. **Chấm:** `grade.sh` — thang 100. **≥ 80 = ĐẠT**, **100 = QUALIFIED**.
> Được dùng mọi nguồn như khi đi làm thật: tài liệu Proxmox, Google, AI, ghi chú của bạn.

## Ticket #4711 — P1

> **Từ:** Trực NOC · **Lúc:** 02:40
>
> App trên **VM 900** không truy cập được từ khoảng 02:00. Tài khoản trực **appops** đăng nhập được nhưng **không thao tác được VM 900**. Tuần trước team hạ tầng có bảo trì cụm và "dọn dẹp cấu hình".
>
> Yêu cầu: khôi phục dịch vụ, đưa hệ thống về trạng thái **được bảo vệ đầy đủ** như thiết kế, và **lấy một bản backup mới của VM 900 trước khi đóng ticket**.

Mọi thứ trong ticket đều là **triệu chứng**, không phải nguyên nhân. Một triệu chứng có thể có nhiều nguyên nhân, và một nguyên nhân có thể gây nhiều triệu chứng.

## Hệ thống bạn tiếp nhận (thiết kế đúng — "trạng thái khỏe")

| Thành phần | Thiết kế |
|---|---|
| VM 900 `app900` | Debian, guest agent bật, disk trên `vmpool` (Ceph), IP `10.0.110.90/24`, VLAN **110** trên `vmbr0`, gateway/DNS `10.0.110.1` (jump) |
| HA | `vm:900` trạng thái `started`, **failback bật**; node-affinity rule **`capstone`**: `pve2:3, pve1:2, pve3:1` — VM chạy ở **pve2** khi pve2 khỏe |
| Mạng node | `vmbr0` VLAN-aware trên `bond0`, `bridge-vids 2-4094` — **mọi** node mang được mọi VLAN tenant |
| Firewall | Bật ở Datacenter và trên VM 900; VM 900 **chặn mặc định cả vào lẫn ra**; chỉ mở: ICMP từ gateway, mọi thứ tới gateway (gồm DNS), TCP 80/443 ra ngoài. Gateway được tham chiếu qua **alias `app_gw`** ở cấp Datacenter |
| Pool & quyền | Pool **`prod`** chứa VM 900 và VM 905. User trực **`appops@pve`** / `AppOps@2026`, role `AppOperator` cấp trên **`/pool/prod`** |
| Backup | Job **`cap-nightly`**: VM 900 + 905 → storage `pbs` (namespace của bạn), 02:00 hằng ngày |

> Storage `pbs` (namespace của bạn) đã kết nối sẵn từ Lab 05.
> Địa chỉ trên là của **cụm 0**. Cụm index N: VLAN = `110 + 500×N`, subnet `10.N.110.0/24` (gateway `.1`, VM 900 `.90`) — grader in đúng giá trị của cụm bạn ở dòng mục.

## Luật chơi — vi phạm thì mục tương ứng bị 0 điểm (grader tự phát hiện)

- **Không tắt firewall** ở bất kỳ tầng nào (Datacenter, VM, `firewall=1` trên NIC), không đổi policy VM 900 thành `ACCEPT`.
- **Không gỡ HA** của VM 900, không sửa rule `capstone` (node, ưu tiên, `strict`), không tắt failback.
- **Không cấp quyền quản trị** cho `appops` (Administrator trên `/`, `Permissions.Modify`, `Sys.Modify`). Quyền phải đúng mức một người trực.
- **Không destroy / restore đè VM 900** — đó là dữ liệu production; backup là để phòng, không phải để "sửa".
- Không đụng `jump`, `pbs0`, Proxmox lớp ngoài.

Hai người một cụm: cùng một ca trực — **thống nhất trước khi gõ**, đổi người cầm bàn phím mỗi khi sửa xong một lỗi.

## Tự chấm — bao nhiêu lần tùy ý

```bash
cd ~/capstone-grader && ./grade.sh
```

Grader **chỉ đọc**, hiện **toàn bộ danh sách mục** và mục nào đã/chưa đạt. Danh sách đó cũng là gợi ý: nó mô tả trạng thái khỏe, không mô tả cách sửa.

| Nhóm | Mục | Điểm |
|---|---|---|
| **A. Mạng ứng dụng (45)** | VM 900 ping được gateway từ trong guest (firewall vẫn bật đủ) | 10 |
| | Jump ping được VM 900 (firewall vẫn bật đủ) | 10 |
| | VM 900 phân giải được DNS | 5 |
| | VLAN 110 có trên `vmbr0` của cả 3 node — cấu hình lưu + đang chạy | 15 |
| | Không còn cơ chế nào tự ghi đè lại cấu hình mạng hỏng | 5 |
| **B. HA / di chuyển (30)** | `vm:900` HA `started`, đang chạy | 10 |
| | Rule `capstone` giữ nguyên, failback còn bật | 5 |
| | VM 900 không còn volume trên storage không-shared | 10 |
| | VM 900 đang chạy trên node ưu tiên `pve2` | 5 |
| **C. Quyền & backup (25)** | `appops@pve` có `VM.PowerMgmt` + `VM.Migrate` trên VM 900, không có quyền quản trị | 15 |
| | PBS có bản backup VM 900 tạo **sau** khi sự cố bắt đầu | 10 |

**Hard gate** (trượt ngay): cụm mất quorum · Ceph `HEALTH_ERR`.

## Tra cứu nhanh (cú pháp — KHÔNG phải trình tự lời giải)

Triage order của khóa: **Network → Storage → Cluster → Proxmox**. Sửa xong một thứ, chạy lại grader — đừng sửa nhiều thứ một lúc rồi không biết cái nào có tác dụng.

**Mạng node & VLAN**

```bash
cat /etc/network/interfaces ; ifreload -a            # cấu hình lưu / áp lại
bridge -c vlan show [dev <port>]                      # VLAN ĐANG CHẠY trên từng cổng
qm config 900 | grep net0                             # tag, firewall=1
journalctl --since "-3h" | grep -iE 'ifreload|netcfg|interfaces'
ls /etc/cron.d/ ; systemctl list-timers               # thứ gì chạy định kỳ trên node?
```

**Trong guest (qua guest agent — không cần mạng)**

```bash
qm agent 900 ping
qm guest exec 900 -- ip -br a                         # xem "exitcode" và "out-data" trong JSON
qm guest exec 900 -- ping -c2 -W2 <ip>
qm guest exec 900 -- getent hosts deb.debian.org
qm terminal 900                                       # console serial (nếu có)
```

**Firewall**

```bash
pve-firewall status ; pve-firewall compile | less     # rule thật sự được áp
cat /etc/pve/firewall/cluster.fw /etc/pve/firewall/900.fw
pvesh get /cluster/firewall/aliases                   # alias cấp Datacenter
pvesh set /cluster/firewall/aliases/<name> --cidr <ip>
```

**HA & migrate**

```bash
ha-manager status ; ha-manager config ; ha-manager rules list
ha-manager set vm:<id> --state <started|stopped|disabled|ignored>
qm migrate <id> <node> --online                      # đọc kỹ thông báo lỗi nếu bị từ chối
qm config <id> --current ; qm pending <id>            # config đang chạy vs thay đổi chờ reboot
```

**Pool, quyền**

```bash
pveum pool list ; pvesh get /cluster/resources --type vm    # cột pool
pveum pool modify <pool> --vms <id>[,<id>] [--delete 1]
pveum acl list ; pveum user permissions <user> --path /vms/<id>
```

**Backup**

```bash
vzdump <id> --storage pbs --mode snapshot              # backup ngay
pvesm list pbs --vmid <id>
```

> Khi grader lên **100/100 = QUALIFIED**. Còn thời gian sau khi ĐẠT (≥ 80): đi tiếp tới 100 — ngoài đời, "gần như khỏe" nghĩa là lần sự cố sau sẽ tệ hơn.
