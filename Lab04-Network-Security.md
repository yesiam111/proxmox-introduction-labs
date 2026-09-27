# Lab 04 (Guided) — Mạng (VLAN, Bond) và Bảo Mật (RBAC, Firewall)

> Hiện thực hóa thiết kế mạng **4 NIC / 3 IP** và siết bảo mật cụm: gộp mgmt+tenant vào **bond0 (active-backup) trên vmbr0 VLAN-aware**, xác nhận jumbo 9000 cho storage, tạo user vận hành RBAC (least privilege), và bật firewall giới hạn truy cập. Đặt nền mạng ổn định cho HA (Bài 05).

> **Phụ lục A (tuỳ chọn, cuối bài):** publish một service ra thẳng mạng vật lý (CT + VM chạy nginx,
> LB hai chân) và live-migrate nó khi đang có traffic.

## Prerequisites

- Đã hoàn tất **guided + challenge Bài 03** (checkpoint 3): cluster quorate, Ceph `HEALTH_OK`, VM/CT nền đã có.
- Topology **4 NIC / 3 IP**: NIC1 (mgmt, đang là vmbr0) + NIC2 (đang trống) → bond0; NIC3 = Ceph (10.0.30, MTU 9000, đặt ở Lab 02); NIC4 = Corosync (10.0.20). Không có LACP ở vSwitch nested → dùng **active-backup** (spec §5).
- Xác định tên NIC bằng `ip -br a` (tên có thể khác ví dụ dưới: NIC1=ens18, NIC2=ens19, NIC3=ens20, NIC4=ens21).
- ⚠️ **Bước 1 đụng mạng MANAGEMENT** (chuyển vmbr0 sang bond) — GIỮ một phiên SSH đang mở và sẵn **console lớp ngoài** phòng tự khóa.

## Quy hoạch (nối tiếp Bài 01/02)

| Mạng | Interface (ví dụ) | Subnet | MTU | Vai trò |
|------|-------------------|--------|-----|---------|
| Management + Tenant | vmbr0 trên bond0 (ens18+ens19) | 10.0.10.0/24 (native) + VLAN tag | 1500 | Web UI/SSH + VM traffic |
| Ceph (public+cluster) | ens20 | 10.0.30.0/24 | **9000** | Client ↔ OSD ↔ OSD |
| Corosync | ens21 | 10.0.20.0/24 | 1500 | Cluster ring (link0) |

---

## Bước 1 — Chuyển mgmt sang bond0 + vmbr0 VLAN-aware (mỗi node)

Gộp NIC1+NIC2 thành `bond0` (active-backup) và cho `vmbr0` chạy trên bond0, bật VLAN-aware để mang cả mgmt (native) lẫn tenant (VLAN tag). **Đây là thao tác đụng đường quản trị** — làm cẩn thận, giữ console.

Trước khi sửa, xác nhận NIC1 (đang là bridge-port của vmbr0) và NIC2 (trống):

```bash
grep -A5 'iface vmbr0' /etc/network/interfaces   # bridge-ports hiện là NIC1 (ens18)
ip -br a                                          # NIC2 (ens19) chưa có IP
```

Sửa `/etc/network/interfaces`: thêm stanza `bond0`, và đổi `vmbr0` để dùng bond0 + VLAN-aware. Kết quả mong muốn (đổi tên ens18/ens19 và `.11` theo máy):

```
auto bond0
iface bond0 inet manual
    bond-slaves ens18 ens19
    bond-mode active-backup
    bond-miimon 100

auto vmbr0
iface vmbr0 inet static
    address 10.0.10.11/24
    gateway 10.0.10.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

Áp dụng và xác minh NGAY (nếu mất mạng → console sửa lại `bridge-ports ens18`):

```bash
ifreload -a
ip -br a | grep 'vmbr0'                                   # vmbr0 vẫn mang 10.0.10.1N
cat /proc/net/bonding/bond0 | grep -E 'Bonding Mode|Currently Active|MII Status'
ssh root@10.0.10.11 true && echo "SSH mgmt còn sống"      # từ phiên mới
```

Kỳ vọng: `vmbr0` giữ IP mgmt; `Bonding Mode: fault-tolerance (active-backup)`, một slave `Currently Active`, MII `up`; SSH mgmt còn vào được.

## Bước 2 — Xác nhận jumbo 9000 (mạng Ceph — đặt ở Lab 02)

MTU 9000 trên NIC3 (Ceph) đã cấu hình ở Lab 02. Xác nhận còn nhất quán end-to-end (DF; 9000 − 28 = 8972):

```bash
ip link show ens20 | grep -o 'mtu 9000'                  # NIC Ceph là jumbo
ping -M do -s 8972 -c3 10.0.30.12 && echo "MTU 9000 pve1->pve2 OK"
ping -M do -s 8972 -c3 10.0.30.13 && echo "MTU 9000 pve1->pve3 OK"
```

Kỳ vọng: cả hai `OK`, `ceph -s` HEALTH_OK. Đây là baseline mà challenge sẽ thay đổi (MTU mismatch).

## Bước 3 — Gán VM vào VLAN + kiểm thử failover bond

Đưa VM 130 (từ Lab 03) vào VLAN 100 trên **vmbr0** (VLAN-aware):

```bash
# `qm set --net0` GHI ĐÈ toàn bộ property: MAC đã sinh, cờ firewall=1, mtu=, queues= đều mất.
# Giữ nguyên phần cũ, chỉ thay tag:
qm set 130 --net0 "$(qm config 130 | sed -n 's/^net0: //p' | sed 's/,tag=[0-9]*//'),tag=100"
qm config 130 | grep net0                    # thấy bridge=vmbr0,tag=100
bridge vlan show dev tap130i0                 # hỏi ĐÚNG port
```

**Đổi IP tĩnh sang dải VLAN 100** (lab không dùng DHCP — đổi subnet thì phải đổi IP):

```bash
qm set 130 --ipconfig0 ip=10.0.100.30/24,gw=10.0.100.1
qm reboot 130                                 # cloud-init áp IP mới + tap replumb sang VLAN 100
sleep 45
qm guest exec 130 -- ping -c1 -W2 10.0.100.1  # gateway VLAN 100 (trên jump)
qm guest exec 130 --timeout 120 -- apt-get update | grep -q '"exitcode" : 0' && echo "APT OK"
# `qm` exit 0 kể cả khi lệnh trong guest fail/timeout — exit code THẬT nằm trong JSON "exitcode".
```

Kỳ vọng: cả hai lệnh trong guest chạy sạch. Từ jump cũng ping được `10.0.100.30` — VM đổi
phân đoạn mạng mà **vẫn phục vụ được**, đó mới là bằng chứng VLAN cấu hình đúng (chỉ nhìn
`bridge vlan show` là chưa đủ).

> Gateway `10.0.100.1` nằm trên `jump` (`ens19.100`). Jump **định tuyến** giữa VLAN 100 và mạng mgmt,
> rồi NAT ra Internet qua `ens18`.

**Vì sao VM ở VLAN 100 vẫn ping được IP của node (`10.0.10.11`) dù khác VLAN?** VLAN chỉ cô lập ở
**tầng 2**. Khác subnet thì guest gửi gói cho gateway `10.0.100.1` (jump); jump có chân ở cả
`10.0.100.0/24` lẫn `10.0.10.0/24`, bật `ip_forward` → nó **định tuyến** giữa hai VLAN (mô hình
*router-on-a-stick*). Khác VLAN ≠ cách ly: muốn cách ly tầng 3 thì phải có rule firewall trên router.

Kiểm thử failover (không làm rớt VM/mgmt): hạ slave đang active của bond0, xác nhận chuyển sang slave kia:

```bash
cat /proc/net/bonding/bond0 | grep 'Currently Active'    # ghi lại slave active (vd ens18)
ip link set ens18 down                                    # hạ slave đó (đổi tên nếu khác)
sleep 1
cat /proc/net/bonding/bond0 | grep 'Currently Active'    # đã chuyển sang ens19; mgmt/VM không rớt
ip link set ens18 up                                      # khôi phục
```

Kỳ vọng: `Currently Active Slave` đổi sang NIC còn lại, MII vẫn `up` — dự phòng hoạt động.

---

## Bước 4 — Firewall: giới hạn Web UI/SSH về mạng quản trị

**Thứ tự an toàn — tạo rule cho phép TRƯỚC, siết default policy SAU.** Giữ một phiên SSH đang mở và sẵn console (outer hypervisor) phòng tự khóa.

> **QUAN TRỌNG:** rule mặc định của Proxmox Firewall đã cho phép **management host** tới 8006 / 22 / 5900–5999 / 3128 / 60000–60050 và **corosync** (UDP 5405–5412 trên cluster network) — mạng cluster được thêm vào IPSet `management` qua alias `cluster_network` (alias tự dò riêng là `local_network` — xem bằng `pve-firewall localnet`). Nhưng firewall **KHÔNG** tự cho phép **Ceph** (MON 6789/3300, OSD/MGR 6800–7300 trên `10.0.30.0/24`) hay **NFS/PBS**. Nếu chỉ dựa vào rule mặc định rồi đặt default DROP, OSD sẽ rớt khỏi cụm và Ceph chuyển HEALTH_WARN/ERR. Vì vậy phải **tin cậy toàn bộ traffic nội cụm** (mạng corosync/storage + IP các node) trước khi siết.
>
> Chạy `pve-firewall localnet` trước để xác nhận dải `local_network` PVE tự dò đúng là `10.0.10.0/24` — nếu sai, IP admin của bạn có thể nằm ngoài và bạn sẽ tự khóa.

Định nghĩa IPSet ở Datacenter (một lần): `management` (dải admin) và `clusternodes` (IP mgmt của 3 node):

```bash
pvesh create /cluster/firewall/ipset --name management
pvesh create /cluster/firewall/ipset/management --cidr 10.0.10.0/24
pvesh create /cluster/firewall/ipset --name clusternodes
for ip in 10.0.10.11 10.0.10.12 10.0.10.13; do
  pvesh create /cluster/firewall/ipset/clusternodes --cidr $ip
done
```

Thêm rule cho từng node `/etc/pve/nodes/<node>/host.fw` — **tin cậy nội cụm, chỉ siết cổng quản trị**:

```bash
# BẮT BUỘC cho CẢ 3 NODE: công tắc ở Datacenter bật host firewall trên mọi node;
# node nào không có host.fw sẽ chạy rule mặc định — vốn KHÔNG cho phép Ceph → OSD rớt.
for n in pve1 pve2 pve3; do cat > /etc/pve/nodes/$n/host.fw <<'EOF'
[OPTIONS]
enable: 1

[RULES]
# Node-to-node (mọi cổng PVE nội bộ: migration, pvecm, proxy...)
IN ACCEPT -source +clusternodes -log nolog
# Toàn bộ traffic trên mạng corosync + Ceph (heartbeat, MON/OSD)
IN ACCEPT -source 10.0.20.0/24 -log nolog
IN ACCEPT -source 10.0.30.0/24 -log nolog
# NFS server (jump) và PBS server
IN ACCEPT -source 10.0.10.1 -log nolog
IN ACCEPT -source 10.0.10.5 -log nolog
# Quản trị: CHỈ mạng mgmt tới Web UI (8006) + SSH (22).
# (Rule mặc định của PVE đã cho phép management host tới 8006/22 — hai dòng này
#  là DƯ THỪA nhưng viết tường minh để học viên thấy rõ ý định và dễ audit.)
IN ACCEPT -source +management -p tcp -dport 8006 -log nolog
IN ACCEPT -source +management -p tcp -dport 22 -log nolog
EOF
done
# Kiem chung TRUOC khi bat cong tac o Datacenter:
for n in pve1 pve2 pve3; do test -s /etc/pve/nodes/$n/host.fw || echo "THIEU host.fw: $n"; done
pve-firewall compile >/dev/null && echo "ruleset compile OK"
```

Chỉ khi cả 3 node đã có `host.fw` mới bật công tắc ở Datacenter.

> **Lưu ý:** chỉ riêng `--enable 1` đã chặn input mặc định trên host — `--policy_in DROP` phía dưới
> gần như chỉ là ghi tường minh. Đừng coi dòng `--enable 1` là bước an toàn còn dòng sau mới nguy hiểm.

```bash
pvesh set /cluster/firewall/options --enable 1
pvesh set /cluster/firewall/options --policy_in DROP
```

Xác minh — TỪ MỘT PHIÊN MỚI trong mạng 10.0.10.0/24, và kiểm cụm không bị lỗi:

```bash
ssh root@10.0.10.11 true && echo "SSH mgmt OK"        # từ mạng mgmt: vào được
pvesh get /cluster/firewall/options --output-format json | grep -o '"enable":[01]'
pvecm status | grep Quorate                            # vẫn Quorate: Yes
sleep 10; ceph -s | grep health                        # PHẢI vẫn HEALTH_OK (OSD không rớt)
ceph osd tree | grep -c 'up'                            # số OSD 'up' không đổi
```

Kỳ vọng: truy cập admin từ mgmt OK; firewall `enable: 1`; **Ceph vẫn HEALTH_OK, OSD vẫn up** (nếu OSD rớt → thiếu rule cho 10.0.20/10.0.30, thêm vào rồi thử lại).

> Nếu mất truy cập: vào console qua outer hypervisor rồi `systemctl stop pve-firewall && pve-firewall stop` (xả rule ngay tại node, **không cần quorum**). Chỉ sửa `/etc/pve/firewall/cluster.fw` hoặc `host.fw` (`enable: 0`) sau khi cụm còn quorum — `/etc/pve` là pmxcfs, mất quorum là read-only. Đây là lý do luôn giữ đường console dự phòng.

> Firewall **giữ nguyên trạng thái bật** từ đây. Bước 5 (SDN) sẽ cho bạn thấy ngay hệ quả của nó:
> một dịch vụ mới chạy trên host — DHCP của SDN — sẽ bị chặn cho tới khi được mở có chủ đích.

---

## Bước 5 — SDN: mạng ảo, IPAM, DHCP và DNS trong Proxmox

> Tới đây mọi IP đều **tĩnh** — đó là chủ ý để lab ổn định và chấm được. Bước này giới thiệu
> **SDN**, nơi Proxmox tự làm gateway, **IPAM**, **DHCP** và **DNS** cho một mạng ảo, tách hẳn
> khỏi hạ tầng đang chạy. Đây là cách Proxmox trả lời câu hỏi "quản lý DHCP/DNS ở đâu?".

### 5.1 Cài dnsmasq (bắt buộc cho DHCP của SDN)

Trên **mỗi node**:

```bash
apt-get install -y dnsmasq
systemctl disable --now dnsmasq     # SDN tự quản lý tiến trình riêng, KHÔNG dùng service mặc định
```

### 5.2 Zone → VNet → Subnet (Datacenter → SDN)

| Lớp | Là gì | Giá trị dùng trong lab |
|---|---|---|
| **Zone** | Kiểu mạng ảo + phạm vi | `labzone`, type **Simple**, ✅ automatic DHCP, IPAM = `pve` |
| **VNet** | "Switch ảo" học viên gắn VM vào | `vnet0`, thuộc `labzone` |
| **Subnet** | Dải IP + gateway + SNAT | `10.0.250.0/24`, gateway `10.0.250.1`, ✅ SNAT |
| **DHCP Range** | Dải cấp động | `10.0.250.50` → `10.0.250.200` |

Làm trên GUI: **Datacenter → SDN → Zones → Add → Simple** (mục Advanced có *automatic DHCP*),
rồi **VNets → Add**, rồi chọn vnet0 → **Subnets → Create** (tab *DHCP Ranges* để nhập dải).

Cuối cùng bấm **SDN → Apply** và chờ task `reload network` xong sạch.

```bash
cat /etc/pve/sdn/zones.cfg /etc/pve/sdn/vnets.cfg /etc/pve/sdn/subnets.cfg
ip -br a show vnet0                 # vnet0 có IP gateway 10.0.250.1
```

### 5.3 Gắn guest vào VNet — và thấy firewall chặn DHCP

Firewall đã bật từ Bước 4 với `policy_in DROP`. DHCP server và DNS của SDN chạy **ngay trên host**:
dnsmasq nghe trên `vnet0`, địa chỉ `10.0.250.1`. Yêu cầu DHCP từ guest vì thế là traffic **đi vào
host** — đúng thứ host firewall đang chặn. Rule mặc định của Proxmox không biết gì về `vnet0`.

Tạo CT xin IP bằng DHCP và quan sát:

```bash
CTTPL=$(pvesm list nfs-store --content vztmpl | awk '/debian-13-standard/{print $1; exit}')
pct create 250 "$CTTPL" --hostname sdn-demo --cores 1 --memory 512 \
  --rootfs vmpool:4 --unprivileged 1 \
  --net0 name=eth0,bridge=vnet0,ip=dhcp
pct start 250
sleep 15
pct exec 250 -- ip -br -4 a show eth0     # KỲ VỌNG: KHÔNG có IPv4 — DHCP bị chặn
```

Chứng minh gói bị chặn ở đâu — trên node, trong khi CT đang xin lease:

```bash
timeout 20 tcpdump -ni vnet0 port 67 or port 68
# Thấy DHCPDISCOVER từ CT đi, nhưng KHÔNG có DHCPOFFER trả về:
# yêu cầu đã tới host rồi bị firewall bỏ trước khi đến dnsmasq.
```

> Không phải lỗi SDN, cũng không phải lỗi CT. Đây là quy luật chung: **mọi dịch vụ mới chạy trên host
> đều bị chặn cho tới khi được mở có chủ đích** — DHCP của SDN chỉ là ví dụ đầu tiên bạn gặp.

### 5.4 Mở firewall cho DHCP và DNS trên `vnet0`

Thêm hai rule ở **Datacenter** (một lần, áp cho `vnet0` của mọi node — Simple zone chạy trên từng
node, nên rule cấp cụm là đúng chỗ). GUI: Datacenter → Firewall → Add.

| Direction | Action | Interface | Macro | Dest |
|---|---|---|---|---|
| in | ACCEPT | `vnet0` | `DHCPfwd` | — |
| in | ACCEPT | `vnet0` | `DNS` | `10.0.250.1` |

Hoặc bằng CLI:

```bash
pvesh create /cluster/firewall/rules --type in --action ACCEPT --iface vnet0 \
  --macro DHCPfwd --enable 1 --comment "SDN DHCP (Lab04 B5)"
pvesh create /cluster/firewall/rules --type in --action ACCEPT --iface vnet0 \
  --macro DNS --dest 10.0.250.1 --enable 1 --comment "SDN DNS (Lab04 B5)"
```

> Đặt **Dest = gateway** cho rule DNS. Bỏ trống là mở mọi traffic DNS, có thể bị lợi dụng để
> lách các rule khác.

Cho CT xin lease lại rồi kiểm:

```bash
pct reboot 250
sleep 15
pct exec 250 -- ip -br -4 a show eth0        # giờ CÓ IP trong dải 10.0.250.50-200
pct exec 250 -- getent hosts deb.debian.org  # DNS do dnsmasq của SDN trả lời
pct exec 250 -- ping -c1 -W2 1.1.1.1         # SNAT ra ngoài — OK
pct exec 250 -- ping -c1 -W2 10.0.250.1      # gateway do SDN tạo — KỲ VỌNG: FAIL
```

Kỳ vọng: có lease, DNS trả lời, `1.1.1.1` ping được — nhưng **ping gateway `10.0.250.1` thì không**.

Nghe ngược đời: ra được tới Internet mà không ping được chính gateway của mình. Lý do là hai gói đi
theo hai đường khác nhau qua host:

| Gói | Đích | Đường đi trong host | Bị policy `DROP` của host chặn? |
|---|---|---|---|
| ping `1.1.1.1` | máy ngoài | **chuyển tiếp** (FORWARD) — vào `vnet0`, SNAT, ra `vmbr0` | Không |
| ping `10.0.250.1` | **chính host** | **đi vào** host (INPUT) | **Có** — chưa có rule cho ICMP trên `vnet0` |

Hai rule ở trên chỉ mở DHCP và DNS. ICMP tới host là một "dịch vụ" nữa, và nó cũng phải được mở có
chủ đích:

```bash
pvesh create /cluster/firewall/rules --type in --action ACCEPT --iface vnet0 \
  --macro Ping --dest 10.0.250.1 --enable 1 --comment "SDN ping gateway (Lab04 B5)"
sleep 3
pct exec 250 -- ping -c1 -W2 10.0.250.1     
```

> Đây là mẫu hình cần nhớ khi chẩn đoán: *"ra được Internet mà không ping được gateway"* gần như luôn
> là **firewall INPUT của gateway**, không phải lỗi định tuyến — nếu định tuyến hỏng thì đã không ra
> được Internet. (Rule này cũng được gỡ ở 5.7 cùng hai rule kia, vì lệnh dọn lọc theo `iface vnet0`.)

**Xem IPAM:** Datacenter → SDN → **IPAM** — bảng lease của mọi guest trong zone. Sửa/đặt trước
mapping được ở đây; sửa xong phải **restart guest từ PVE** (restart bên trong guest không đủ).

### 5.5 DNS tuỳ chỉnh cho VNet

**Ghi lại trạng thái TRƯỚC khi sửa** — để lát nữa thấy tận mắt DHCP đổi nó:

```bash
pct exec 250 -- cat /etc/resolv.conf      # ghi lại dòng nameserver (thường là gateway 10.0.250.1)
```

Muốn VNet phát DNS server riêng, sửa `/etc/pve/sdn/subnets.cfg`:

```
subnet: labzone-10.0.250.0-24
	vnet vnet0
	dhcp-range start-address=10.0.250.50,end-address=10.0.250.200
	dhcp-dns-server 10.0.10.1
	gateway 10.0.250.1
	snat 1
```

**Apply lại SDN**, rồi cho CT **xin lease mới** — tuỳ chọn DHCP (như DNS server) chỉ đến client
khi có lease mới, sửa cấu hình xong chưa đủ:

```bash
pct reboot 250 && sleep 15
pct exec 250 -- cat /etc/resolv.conf           # nameserver giờ là 10.0.10.1 — ĐỔI so với lúc trước
pct exec 250 -- getent hosts pbs0.lab.local    # phải ra 10.0.10.5
```

> Chính sự **thay đổi** của dòng `nameserver` mới là bằng chứng DHCP đã apply — chỉ nhìn giá trị sau
> thì chưa đủ, vì `10.0.10.1` cũng là DNS của node, và với CT không đặt `nameserver` riêng, Proxmox có
> thể chép DNS của host vào. File do Proxmox viết mở đầu bằng `# --- BEGIN PVE ---`; không có dòng đó
> thì là DHCP client viết. Nếu lúc trước đã là `10.0.10.1` sẵn thì bước này không chứng minh được gì —
> thử lại với một giá trị khác, ví dụ `dhcp-dns-server 1.1.1.1` (lúc đó `pbs0` sẽ **không** resolve được,
> cũng là một kết quả đáng xem).

> Vì sao hỏi `pbs0` chứ không phải `pve1`? `pbs0.lab.local` là tên **chỉ DNS của lab biết** — DNS
> công cộng không trả lời được — nên resolve được nghĩa là CT đang hỏi đúng DNS lab. Còn tên node
> (`pve1`, ...) **không** nằm trong DNS của jump: mọi cụm trong lớp dùng chung tên `pve1/2/3` với IP
> khác nhau, nên tên node chỉ sống trong `/etc/hosts` của từng node (Lab 01). Chạy
> `getent hosts pve1.lab.local` **trên node** vẫn ra kết quả — nhưng là từ `/etc/hosts` cục bộ, không
> phải từ DNS.

### 5.6 Các loại Zone — lựa chọn như thế nào?

| Zone | Phạm vi | Dùng khi |
|---|---|---|
| **Simple** | **trong một node** (mỗi node một instance) | mạng NAT cục bộ, lab, DMZ nhỏ — **dùng ở bước này** |
| **VLAN** | toàn cụm, dựa trên VLAN sẵn có | multi-tenant với hạ tầng VLAN có sẵn — chính là Bước 1/3 nhưng quản lý tập trung |
| **QinQ** | toàn cụm, VLAN lồng VLAN | nhà cung cấp dịch vụ, chồng VLAN khách hàng |
| **VXLAN** | toàn cụm, overlay L2 qua L3 | trải mạng qua nhiều site/subnet |
| **EVPN** | toàn cụm, overlay + routing | multi-tenant có định tuyến L3, exit-node ra ngoài |

> Simple zone là **zone cục bộ**: VM di trú sang node khác vẫn vào `vnet0` của node đó và giữ IP
> nhờ IPAM cấp cụm, nhưng gateway/dnsmasq là của node mới. Muốn L2 thật sự trải toàn cụm thì
> dùng **VLAN** hoặc **VXLAN** zone.

### 5.7 Dọn dẹp (bắt buộc trước checkpoint)

`vnet0` và CT 250 **không** thuộc thiết kế nền của khóa — gỡ để checkpoint và challenge không bị nhiễu:

```bash
pct stop 250 && pct destroy 250 --purge
qm set 130 --delete net1                   # gỡ NIC phụ trên OVS bridge
# SDN: Datacenter -> SDN -> xoá Subnet -> VNet -> Zone -> Apply

# Gỡ các rule firewall của vnet0 (DHCP, DNS, Ping) — nếu không, cluster.fw giữ mãi rule trỏ tới interface đã chết.
# Xoá từ vị trí CAO xuống THẤP: xoá một rule thì các rule phía sau bị đôn số.
for pos in $(pvesh get /cluster/firewall/rules --output-format json | python3 -c '
import sys, json
rs = [r for r in json.load(sys.stdin) if r.get("iface") == "vnet0"]
print(" ".join(str(r["pos"]) for r in sorted(rs, key=lambda r: -r["pos"])))'); do
  pvesh delete /cluster/firewall/rules/$pos
done
grep -q vnet0 /etc/pve/firewall/cluster.fw && echo "CÒN rule vnet0" || echo "rule vnet0 đã gỡ"
```

---

## Bước 6 — RBAC: user vận hành + group + ACL (least privilege)

Tạo group vận hành VM, user `ops`, và gán role theo path (chạy một lần trên pve1 — đồng bộ cluster):

```bash
pveum group add vmops --comment "Van hanh VM/CT"
pveum user add ops@pve --password 'Ops@2026'
pveum user modify ops@pve --group vmops
pveum acl modify /vms --group vmops --role PVEVMAdmin
pveum acl modify /storage/vmpool --group vmops --role PVEDatastoreUser
```

Xác minh:

```bash
pveum acl list | grep vmops                  # thấy ACL tại /vms và /storage/vmpool
pveum user permissions ops@pve --path /vms    # có quyền VM.* tại /vms
```

Kỳ vọng: user `ops@pve` có quyền quản lý VM tại `/vms`, không có quyền hạ tầng (cluster/network).

> Từ giờ, thao tác VM hằng ngày dùng `ops@pve` (đăng nhập Web UI hoặc `pvesh` với token) — giữ `root@pam` cho cứu hộ. Nên bật 2FA cho `ops@pve` qua Web UI (User → TFA → TOTP).

---

## Bước 7 — Xác nhận cuối (chạy trên pve1)

```bash
echo "== MTU storage =="
ping -M do -s 8972 -c1 10.0.30.12 >/dev/null 2>&1 && ping -M do -s 8972 -c1 10.0.30.13 >/dev/null 2>&1 && echo "MTU 9000 OK" || echo "MTU FAIL"
echo "== bond =="
grep -q 'active-backup' /proc/net/bonding/bond0 && echo "bond OK" || echo "bond FAIL"
echo "== vmbr0 vlan-aware =="
grep -qE '^[[:space:]]*bridge-vlan-aware[[:space:]]+yes' /etc/network/interfaces && echo "vlan-aware OK" || echo "vlan-aware FAIL"
echo "== VM130 VLAN =="
qm config 130 | grep -q 'tag=100' && echo "VM VLAN OK" || echo "VM VLAN FAIL"
echo "== RBAC =="
pveum acl list | grep -q 'vmops' && echo "RBAC OK" || echo "RBAC FAIL"
echo "== SDN đã dọn (Bước 5.7) =="
! ip link show vnet0 >/dev/null 2>&1 && ! grep -q vnet0 /etc/pve/firewall/cluster.fw \
  && echo "SDN cleanup OK" || echo "SDN cleanup FAIL — còn vnet0 hoặc rule firewall của nó"
echo "== firewall =="
pvesh get /cluster/firewall/options --output-format json 2>/dev/null | grep -q '"enable":1' && echo "firewall OK" || echo "firewall FAIL"
echo "== nền =="
pvecm status | grep -q 'Quorate: *Yes' && ceph health | grep -q HEALTH_OK && echo "cluster OK" || echo "cluster FAIL"
```

Kỳ vọng: tất cả `OK`.

---

# Phụ lục A (TUỲ CHỌN) — Publish một service ra mạng ngoài

> **Không thuộc checkpoint 4.** Bước 7 ở trên vẫn là điều kiện duy nhất để nhận challenge —
> cụm không có IP ngoài vẫn hoàn thành Bài 04 bình thường. Phụ lục này chạy khi lớp **có
> IP ngoài dư và còn thời gian** (~45 phút).
>
> **Lý do nên làm:** suốt khoá, mọi thứ chỉ "đến được" qua jump host. Ở đây bạn đưa một
> service ra **thẳng mạng vật lý**, rồi **live migrate nó trong lúc đang có traffic thật** —
> đó là bằng chứng cuối cùng rằng thiết kế mạng + HA của bạn dùng được, không chỉ "cấu hình đúng".

## A.0 Điều kiện (INSTRUCTOR chuẩn bị trước)

Topology nền **4 NIC / 3 IP không đổi**. Phần dưới là **đường phụ**, chỉ thêm trên cụm nào làm phụ lục này.

Trên **host lớp ngoài**, thêm vNIC thứ 5 cho cả 3 node, nối vào bridge LAN:

```bash
# firewall=0 BẮT BUỘC: PVE lọc MAC bằng ebtables, sẽ chặn MAC của guest bên trong
for id in 101 102 103; do qm set $id --net4 virtio,bridge=vmbr0,firewall=0; done
```

Trong **mỗi node nested**, biến NIC đó thành bridge trung chuyển (KHÔNG đặt IP):

```
auto vmbr9
iface vmbr9 inet manual
    bridge-ports ens22          # NIC thứ 5 — xác định bằng `ip -br a`
    bridge-stp off
    bridge-fd 0
```

Cấp cho học viên **1 IP ngoài** (vd `192.168.1.110/24`, gw `192.168.1.1`).

Kiểm tra:

```bash
ip -br link show vmbr9          # phải UP trên cả pve1/pve2/pve3
```

## A.1 CT 430 — backend nginx trên VLAN app

```bash
CTTPL=$(pvesm list nfs-store --content vztmpl | awk '/debian-12-standard/{print $1; exit}')
pct create 430 "$CTTPL" \
  --hostname web-ct --cores 1 --memory 512 \
  --rootfs vmpool:4 --unprivileged 1 \
  --net0 name=eth0,bridge=vmbr0,tag=100,ip=10.0.100.43/24,gw=10.0.100.1 \
  --nameserver 10.0.10.1 --searchdomain lab.local
pct start 430
sleep 10
pct exec 430 -- bash -lc 'apt-get update -qq && apt-get install -y nginx >/dev/null'
pct exec 430 -- bash -lc 'echo "backend: CT 430 (container)" > /var/www/html/index.html'
pct exec 430 -- curl -s localhost        # phải in ra dòng trên
```

## A.2 VM 130 — backend thứ hai (đã có từ Bài 03)

Cùng một pool backend có **cả CT lẫn VM** — đúng câu hỏi "khi nào CT, khi nào VM" của Bài 03.

```bash
qm guest exec 130 --timeout 180 -- bash -lc \
  'apt-get update -qq && apt-get install -y nginx >/dev/null; echo "backend: VM 130 (virtual machine)" > /var/www/html/index.html'
pct exec 430 -- curl -s 10.0.100.30      # từ CT gọi sang VM — phải ra dòng của VM 130
```

## A.3 VM 440 — load balancer, hai chân

Clone từ template 9000 (Bài 03), gắn **hai** NIC:

| Chân | Bridge | VLAN | IP | Gateway |
|---|---|---|---|---|
| `net0` backend | `vmbr0` | tag 100 | `10.0.100.44/24` | **không** |
| `net1` frontend | `vmbr9` | — | `192.168.1.110/24` | `192.168.1.1` |

```bash
qm clone 9000 440 --name lb --full
qm set 440 --net0 virtio,bridge=vmbr0,tag=100
qm set 440 --net1 virtio,bridge=vmbr9
qm set 440 --ipconfig0 ip=10.0.100.44/24
qm set 440 --ipconfig1 ip=192.168.1.110/24,gw=192.168.1.1
qm set 440 --memory 1024 --cores 1 --agent 1
qm start 440
```

> **Chỉ đặt gateway trên chân frontend.** Hai default gateway sẽ gây định tuyến bất đối xứng —
> lúc chạy lúc không, và cực kỳ khó chẩn đoán. Subnet backend `10.0.100.0/24` là directly-connected
> nên không cần gateway.

Cài nginx làm reverse proxy:

```bash
qm guest exec 440 --timeout 180 -- bash -lc 'apt-get update -qq && apt-get install -y nginx >/dev/null' \
  | grep -q '"exitcode" : 0' || echo "CÀI NGINX THẤT BẠI trong guest — kiểm NAT/DNS trước khi đi tiếp"
qm guest exec 440 -- bash -lc 'cat > /etc/nginx/sites-available/default <<EOF
upstream app {
    server 10.0.100.43:80;   # CT 430
    server 10.0.100.30:80;   # VM 130
}
server {
    listen 80;
    location / {
        proxy_pass http://app;
        add_header X-Served-By \$upstream_addr always;
    }
}
EOF
nginx -t && systemctl reload nginx'
```

## A.4 Mở firewall cho port đã publish

Bước 4 đã siết cụm lại. Service mới phải được mở **có chủ đích** — đây chính là quy trình thật:

```bash
# Nếu đã bật firewall trên NIC của VM 440 thì thêm rule cho tcp/80:
cat >> /etc/pve/firewall/440.fw <<'EOF'
[OPTIONS]
enable: 1

[RULES]
IN ACCEPT -p tcp -dport 80 -log nolog
EOF
pve-firewall compile >/dev/null && echo "firewall 440 OK"
```

## A.5 Nghiệm thu — từ máy thật, không qua jump

Từ laptop của bạn trên mạng vật lý:

```bash
curl -s http://192.168.1.110/            # lần lượt ra CT 430 rồi VM 130 (round-robin)
curl -sI http://192.168.1.110/ | grep X-Served-By
```

Kỳ vọng: gọi nhiều lần thì **luân phiên** giữa hai backend. Đây là lần đầu trong khoá bạn chạm
tới workload **không qua jump host**.

## A.6 Điểm nhấn — live migration khi đang có traffic

```bash
# TERMINAL 1 (laptop): bắn liên tục, đếm request lỗi
fail=0; for i in $(seq 1 600); do
  curl -sf -m 2 -o /dev/null http://192.168.1.110/ || fail=$((fail+1))
  sleep 0.2
done; echo "FAILED: $fail / 600"

# TERMINAL 2 (pve1): migrate LB sang node khác trong lúc terminal 1 đang chạy
qm migrate 440 pve2 --online
```

Kỳ vọng: **0–1 request lỗi** trên 600. VM giữ nguyên MAC và IP, `vmbr9` có mặt ở mọi node nên
chân frontend nối lại ngay sau khi tap được replumb.

> Nếu **mọi** request lỗi sau migrate: node đích thiếu `vmbr9` (xem A.0).
> Nếu lỗi kéo dài vài giây: bình thường — switch vật lý cần học lại MAC; gửi gratuitous ARP
> (`arping -U -I <iface> <ip>` trong guest) sẽ rút ngắn.

Đưa VM 440 vào HA rồi lặp lại với `ha-manager crm-command migrate vm:440 pve3` nếu đã học Bài 05.

## A.7 Dọn dẹp (nếu không giữ cho capstone)

```bash
qm stop 440 && qm destroy 440 --purge
pct stop 430 && pct destroy 430 --purge
```
