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
bridge vlan show | grep -A1 tap130            # tap của VM 130 gắn VLAN 100
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

> Gateway `10.0.100.1` nằm trên `jump` (`ens19.100`) và được NAT ra Internet
> Nếu ping gateway fail mà `bridge vlan show` vẫn đúng:
> kiểm trunk VLAN ở lớp ngoài, không phải cấu hình trong node.

Kiểm thử failover (không làm rớt VM/mgmt): hạ slave đang active của bond0, xác nhận chuyển sang slave kia:

```bash
cat /proc/net/bonding/bond0 | grep 'Currently Active'    # ghi lại slave active (vd ens18)
ip link set ens18 down                                    # hạ slave đó (đổi tên nếu khác)
sleep 1
cat /proc/net/bonding/bond0 | grep 'Currently Active'    # đã chuyển sang ens19; mgmt/VM không rớt
ip link set ens18 up                                      # khôi phục
```

Kỳ vọng: `Currently Active Slave` đổi sang NIC còn lại, MII vẫn `up` — dự phòng hoạt động.

## Bước 4 — Open vSwitch (vSwitch thay cho Linux bridge)

> **OVS là LỰA CHỌN THAY THẾ, không phải bổ sung.** Tài liệu Proxmox nói rõ: **không được trộn**
> OVS với bridge/bond/VLAN của Linux. Một node đã dùng `bond0` + `vmbr0` Linux (Bước 1) thì
> muốn chuyển sang OVS phải chuyển **cả cụm interface đó**.
>
> Vì vậy lab này làm **OVS trên một bridge riêng, không có NIC vật lý** — học đúng cú pháp và
> công cụ mà **không đụng đường quản trị**. Cấu hình chuyển đổi thật cho production ở cuối bước.

### 4.1 Cài đặt và xem cấu trúc

```bash
apt-get install -y openvswitch-switch
ovs-vsctl show                      # rỗng lúc đầu — đây là "switch" của bạn
ovs-vsctl --version
```

### 4.2 Tạo OVS bridge + OVS internal port

Thêm vào `/etc/network/interfaces` (đổi `.11` theo node):

```
# OVS bridge dùng để học — KHÔNG gắn NIC vật lý, không ảnh hưởng mgmt
auto vmbr9
iface vmbr9 inet manual
    ovs_type OVSBridge
    ovs_ports ovsint200

# Host interface trong VLAN 200 (tương đương "RVI/IRB" trên switch vật lý)
auto ovsint200
iface ovsint200 inet static
    ovs_type OVSIntPort
    ovs_bridge vmbr9
    ovs_options tag=200
    address 10.0.200.11/24
```

```bash
ifreload -a
ovs-vsctl show                      # thấy Bridge vmbr9, Port ovsint200 tag: 200
ip -br a show ovsint200             # có IP 10.0.200.11/24
```

> ⚠️ **Bẫy  #1 của OVS:** mọi interface thành viên **phải được liệt kê trong `ovs_ports`** của
> bridge, *dù* bản thân nó đã khai `ovs_bridge`. Thiếu dòng đó thì interface không được bật —
> không báo lỗi rõ ràng. Đây là lỗi hay gặp nhất khi mới dùng OVS.

### 4.3 Gắn VM vào OVS bridge

```bash
qm set 130 --net1 virtio,bridge=vmbr9,tag=200      # NIC phụ, không đụng net0 đang chạy
qm config 130 | grep net1
ovs-vsctl show                                      # xuất hiện Port tap130i1 tag: 200
```

> **Khác biệt mô hình:** Linux bridge kiểu cũ cần **một bridge cho mỗi VLAN**. OVS dùng **một
> bridge duy nhất mang mọi VLAN**, tag đặt trên từng cổng VM. Thêm/bớt VLAN không phải tạo bridge mới.

### 4.4 So sánh và tiêu chí chọn

| | Linux bridge (mặc định PVE) | Open vSwitch |
|---|---|---|
| Cài đặt | có sẵn | `openvswitch-switch` |
| VLAN | `bridge-vlan-aware yes` + `tag=` | một bridge mang mọi VLAN, `tag=` trên cổng |
| Bond | `bond-mode` (kernel bonding) | `OVSBond` + `ovs_options bond_mode=` |
| IP cho host trên VLAN | sub-interface `vmbr0.50` | **OVSIntPort** |
| MTU | `mtu 9000` | **`ovs_mtu 9000`** trên NIC + bond + bridge |
| Tính năng thêm | — | RSTP, VXLAN, OpenFlow, `bond_mode=balance-slb` (không cần switch hỗ trợ LACP) |
| Công cụ | `bridge`, `ip` | `ovs-vsctl`, `ovs-appctl` |

**Khi nào chọn OVS:** cần RSTP, OpenFlow, hoặc bond chia tải mà switch **không** hỗ trợ LACP
(`balance-slb`). Ngoài các nhu cầu đó, Linux bridge đơn giản hơn và là mặc định của Proxmox.

### 4.5 (Tham chiếu) Cấu hình production tương đương Bước 1 bằng OVS

Không chạy trong lab — đây là bản dịch của thiết kế bond + VLAN sang OVS để mang về dùng:

```
auto ens18
iface ens18 inet manual
    ovs_mtu 1500

auto ens19
iface ens19 inet manual
    ovs_mtu 1500

auto bond0
iface bond0 inet manual
    ovs_bridge vmbr0
    ovs_type OVSBond
    ovs_bonds ens18 ens19
    ovs_options bond_mode=balance-slb vlan_mode=native-untagged
    # LACP (cần switch cấu hình tương ứng):
    # ovs_options bond_mode=balance-tcp lacp=active other_config:lacp-time=fast

auto vmbr0
iface vmbr0 inet manual
    ovs_type OVSBridge
    ovs_ports bond0 mgmt

auto mgmt
iface mgmt inet static
    ovs_type OVSIntPort
    ovs_bridge vmbr0
    ovs_options vlan_mode=access
    address 10.0.10.11/24
    gateway 10.0.10.1
```

> ⚠️ **Bẫy số 2:** với MTU > 1500, OVS dùng **`ovs_mtu`** (không phải `mtu`) và phải đặt trên
> **cả** NIC vật lý, bond **và** bridge — thiếu một chỗ là interface không lên.
>
> ⚠️ **Chuyển đổi thật luôn cần console.** Đổi `vmbr0` sang OVS là cắt đường mgmt trong lúc
> `ifreload` chạy. Làm qua noVNC của lớp ngoài, không qua SSH.

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

### 5.3 Mở firewall cho DHCP và DNS (phối hợp với Bước 7)

Vì cụm đã bật firewall, **phải** cho phép DHCP + DNS trên `vnet0`, nếu không guest không xin được
lease. Datacenter → Firewall → Add, hai rule:

| Direction | Action | Interface | Macro | Dest |
|---|---|---|---|---|
| in | ACCEPT | `vnet0` | `DHCPfwd` | — |
| in | ACCEPT | `vnet0` | `DNS` | `10.0.250.1` |

> Đặt **Dest = gateway** cho rule DNS. Bỏ trống là mở mọi traffic DNS, có thể bị lợi dụng để
> lách các rule khác.

### 5.4 Gắn guest vào VNet và xem DHCP hoạt động

```bash
# CT mới, lấy IP bằng DHCP từ SDN
CTTPL=$(pvesm list nfs-store --content vztmpl | awk '/debian-12-standard/{print $1; exit}')
pct create 250 "$CTTPL" --hostname sdn-demo --cores 1 --memory 512 \
  --rootfs vmpool:4 --unprivileged 1 \
  --net0 name=eth0,bridge=vnet0,ip=dhcp
pct start 250
sleep 10
pct exec 250 -- ip -br a show eth0        # IP trong dải 10.0.250.50-200
pct exec 250 -- ping -c1 10.0.250.1       # gateway do SDN tạo
pct exec 250 -- getent hosts deb.debian.org   # DNS do dnsmasq của SDN trả lời
pct exec 250 -- ping -c1 -W2 1.1.1.1      # SNAT ra ngoài
```

**Xem IPAM:** Datacenter → SDN → **IPAM** — bảng lease của mọi guest trong zone. Sửa/đặt trước
mapping được ở đây; sửa xong phải **restart guest từ PVE** (restart bên trong guest không đủ).

### 5.5 DNS tuỳ chỉnh cho VNet

dnsmasq mặc định dùng DNS của host. Muốn chỉ định riêng, sửa `/etc/pve/sdn/subnets.cfg`:

```
subnet: labzone-10.0.250.0-24
	vnet vnet0
	dhcp-range start-address=10.0.250.50,end-address=10.0.250.200
	dhcp-dns-server 10.0.10.1
	gateway 10.0.250.1
	snat 1
```

Apply lại SDN rồi `pct exec 250 -- getent hosts pve1.lab.local` để xác nhận đã hỏi đúng DNS lab.

### 5.6 Các loại Zone — chọn cái nào

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

`vnet0` và CT 250 **không** thuộc thiết kế nền của khóa — gỡ để Bước 6 và challenge không bị nhiễu:

```bash
pct stop 250 && pct destroy 250 --purge
qm set 130 --delete net1                   # gỡ NIC phụ trên OVS bridge
# SDN: Datacenter -> SDN -> xoá Subnet -> VNet -> Zone -> Apply
```

`vmbr9`/`ovsint200` giữ lại được (không có NIC vật lý, vô hại), hoặc xoá khỏi
`/etc/network/interfaces` rồi `ifreload -a` cho sạch.

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

## Bước 7 — Firewall: giới hạn Web UI/SSH về mạng quản trị

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

Xác minh — TỪ MỘT PHIÊN MỚI trong mạng 10.0.10.0/24, và kiểm cụm không vỡ:

```bash
ssh root@10.0.10.11 true && echo "SSH mgmt OK"        # từ mạng mgmt: vào được
pvesh get /cluster/firewall/options --output-format json | grep -o '"enable":[01]'
pvecm status | grep Quorate                            # vẫn Quorate: Yes
sleep 10; ceph -s | grep health                        # PHẢI vẫn HEALTH_OK (OSD không rớt)
ceph osd tree | grep -c 'up'                            # số OSD 'up' không đổi
```

Kỳ vọng: truy cập admin từ mgmt OK; firewall `enable: 1`; **Ceph vẫn HEALTH_OK, OSD vẫn up** (nếu OSD rớt → thiếu rule cho 10.0.20/10.0.30, thêm vào rồi thử lại).

> Nếu mất truy cập: vào console qua outer hypervisor rồi `systemctl stop pve-firewall && pve-firewall stop` (xả rule ngay tại node, **không cần quorum**). Chỉ sửa `/etc/pve/firewall/cluster.fw` hoặc `host.fw` (`enable: 0`) sau khi cụm còn quorum — `/etc/pve` là pmxcfs, mất quorum là read-only. Đây là lý do luôn giữ đường console dự phòng.

## Bước 8 — Xác nhận cuối (chạy trên pve1)

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
echo "== firewall =="
pvesh get /cluster/firewall/options --output-format json 2>/dev/null | grep -q '"enable":1' && echo "firewall OK" || echo "firewall FAIL"
echo "== nền =="
pvecm status | grep -q 'Quorate: *Yes' && ceph health | grep -q HEALTH_OK && echo "cluster OK" || echo "cluster FAIL"
```

Kỳ vọng: tất cả `OK`.

---

# Phụ lục A (TUỲ CHỌN) — Publish một service ra mạng ngoài

> **Không thuộc checkpoint 4.** Bước 8 ở trên vẫn là điều kiện duy nhất để nhận challenge —
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

Bước 7 vừa siết cụm lại. Service mới phải được mở **có chủ đích** — đây chính là quy trình thật:

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