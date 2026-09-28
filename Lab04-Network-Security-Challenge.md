# Lab 04 — Challenge: "Cluster Chậm Không Rõ Nguyên Nhân"

> **Kịch bản:** Sáng nay các nhóm quan sát thấy cụm "chậm": Ceph lúc nhanh lúc chậm, backup/migration ì ạch, và một VM mới triển khai (VM 420) không thông mạng với các máy cùng VLAN. Không có cảnh báo rõ ràng, log rời rạc. Nhiệm vụ: **chẩn đoán và sửa** đến khi mạng storage đạt hiệu năng bình thường và VM 420 về đúng VLAN.
>
> Lưu ý: **một triệu chứng có thể có nhiều hơn một nguyên nhân.** Sửa xong một chỗ mà cụm vẫn chưa
> khỏe thì chưa xong. Và **firewall phải giữ bật** — kể cả firewall của VM. Gỡ firewall để "hết lỗi"
> không phải là sửa.
>
> **Chấm:** `grade04.sh` từ jump host — kết quả **PASS/FAIL**, PASS khi **tất cả** tiêu chí đạt (kể cả điều kiện nền).

## Tên tài nguyên BẮT BUỘC

| Tài nguyên | Giá trị |
|-----------|---------|
| VM cần sửa VLAN | VMID **420** |
| VLAN đúng của VM 420 | **100** (trên `vmbr0`) |
| Mạng storage phải đạt | MTU **9000** nhất quán (mạng `10.0.30.0/24`) |

## Acceptance criteria

Grader kiểm chính trên `pve1`:

1. **Điều kiện nền:** cluster quorate; `ceph health` = `HEALTH_OK`.
2. **Jumbo end-to-end:** `ping -M do -s 8972` THÀNH CÔNG giữa **mọi cặp node, theo cả hai chiều**.
3. **MTU nhất quán, bền vững:** interface mạng storage của **cả 3 node** ở `mtu 9000` — cả lúc đang chạy lẫn trong cấu hình (sống sót qua `ifreload`/reboot).
4. **Firewall vẫn bật:** Datacenter firewall bật; NIC của VM 420 vẫn có `firewall=1`; firewall của VM 420 bật và `policy_in` vẫn là `DROP` — traffic cần thiết phải được mở bằng **rule**, không phải bằng cách nới policy.
5. **VLAN đúng trong cấu hình:** VM 420 đang chạy, `net0` trên `vmbr0` với `tag=100`.
6. **VLAN đúng trong kernel:** port của VM 420 trên `vmbr0` **thực sự** nằm ở VLAN 100 — không chỉ trong cấu hình.
7. **Guest thông mạng thật:** từ jump ping được VM 420 tại `10.0.100.42`.

> `iperf3` **không** được chấm — grader chỉ in số đo để tham khảo. MTU được chấm dứt khoát bằng DF-ping.

## Tra cứu nhanh

**Đo đường mạng**

```bash
ping -M do -s <payload> -c3 <ip>     # DF: gói KHÔNG được phân mảnh. payload = MTU - 28
                                     #   (trừ 20B header IP + 8B header ICMP) -> MTU 9000 = 8972
iperf3 -s -1 -D                      # phía nhận: chạy nền, phục vụ đúng 1 kết nối rồi thoát
iperf3 -c <ip> -t 5                  # phía gửi
```

> DF-ping trả lời câu hỏi "đường này chịu được gói lớn tới đâu", `iperf3` trả lời "băng thông bao nhiêu".
> **Hướng quan trọng:** MTU chỉ giới hạn chiều gửi. Node đặt nhầm MTU nhỏ vẫn nhận được gói lớn và trả lời
> bằng gói phân mảnh, nên ping **tới** nó vẫn thành công — chỉ ping **từ** nó mới lộ lỗi.
> MTU lệch thường cho DF-ping FAIL nhưng throughput *vẫn trông bình thường* (do TCP tự hạ MSS).

**Interface & MTU**

```bash
ip -br addr                          # tóm tắt: interface nào mang IP nào
ip link show <dev>                   # MTU đang chạy THỰC TẾ
ip link set <dev> mtu 9000           # chỉ có hiệu lực tới lần reboot/ifreload kế tiếp
```

Persistent thì phải nằm trong `/etc/network/interfaces` (dòng `mtu 9000` trong đúng stanza `iface`):

```bash
grep -A8 'iface <dev>' /etc/network/interfaces
ifreload -a                          # áp lại file cấu hình (thay cho restart networking)
ifquery -a                           # xem PVE hiểu file cấu hình ra sao
cat /proc/net/bonding/bond0          # chế độ bond + trạng thái slave
```

**VLAN của guest**

```bash
qm config <vmid> | grep ^net         # bridge nào, tag bao nhiêu
qm set <vmid> --net0 virtio,bridge=vmbr0,tag=<vlan>
bridge vlan show dev tap<vmid>i0     # VLAN mà kernel THỰC SỰ gán cho tap của VM
bridge link                          # tap đang cắm vào bridge nào
```

> `qm set --net0` **ghi đè toàn bộ** property net0: MAC đã sinh, cờ `firewall=`, `mtu=` đều mất
> nếu bạn không gõ lại. Đổi tag khi VM đang chạy chưa replumb tap — phải stop/start VM.
>
> **NIC có `firewall=1` thì đường đi khác:** `tap<vmid>i0` cắm vào một bridge riêng `fwbr<vmid>i0`,
> và port nối sang `vmbr0` là `fwpr<vmid>p0` — **VLAN nằm trên port đó**, không phải trên tap:
>
> ```bash
> bridge link | grep <vmid>            # xem tap / fwbr / fwpr của VM nằm ở đâu
> bridge vlan show dev fwpr<vmid>p0    # VLAN thật của VM khi firewall=1
> ```

**Firewall (nếu nghi rule chặn)** — ba tầng: Datacenter, node (host), **VM**

```bash
pve-firewall status ; pve-firewall compile ; pve-firewall localnet
cat /etc/pve/firewall/cluster.fw              # Datacenter
cat /etc/pve/nodes/<node>/host.fw             # node
cat /etc/pve/firewall/<vmid>.fw               # VM — chỉ có hiệu lực khi NIC có firewall=1
pvesh get /nodes/<node>/qemu/<vmid>/firewall/options
pvesh get /nodes/<node>/qemu/<vmid>/firewall/rules
```

Cú pháp rule trong file `.fw` (một dòng một rule, nằm dưới `[RULES]`):

```
IN  ACCEPT -p tcp -dport 22 -source <ip|cidr> -log nolog
IN  Ping(ACCEPT) -log nolog                  # macro: cho phép ICMP echo
```

> Manh mối hữu ích: **đi RA được nhưng không ai vào được** gần như luôn là firewall chiều vào
> (`policy_in`), vì gói trả lời của kết nối đi ra được firewall coi là `ESTABLISHED` và cho qua.

> Mất truy cập: vào console từ hypervisor lớp ngoài rồi `systemctl stop pve-firewall && pve-firewall stop`
> — xả rule ngay tại node, **không cần quorum** (`/etc/pve` là pmxcfs, mất quorum là read-only).

**Ceph — ai đang kêu, và kêu gì**

```bash
ceph -s                              # tổng quan
ceph health detail                   # từng cảnh báo cụ thể (OSD nào down, PG nào degraded…)
ceph osd tree                        # OSD nào up/down, nằm trên host nào
journalctl -u ceph-osd@<id> -n 50    # log của một OSD (chạy trên node chứa OSD đó)
```

**Điều kiện nền**

```bash
pvecm status | grep Quorate ; ceph health
```

## Cách tự kiểm

```bash
cd ~/graders && ./grade04.sh
```
