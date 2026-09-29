# Lab 06a (Guided) — Giám sát, Báo cáo và Baseline Hiệu năng

> Phần hands-on của "Giám sát và tối ưu hóa hiệu suất" (Bài 06), chạy **trước capstone**.
> Mục tiêu: cài & cấu hình được một công cụ giám sát, xuất được báo cáo hệ thống, và xác lập
> **baseline** để về sau biết "chậm" là chậm so với cái gì.
> **Chấm điểm:** không chấm — cuối bài có khối xác nhận phải chạy sạch.

## Prerequisites

- Cụm 3 node quorate, Ceph `HEALTH_OK` (sau checkpoint 5).
- `jump` (`10.0.10.1`) truy cập được từ node.

---

## Bước 1 — Giám sát bằng GUI và CLI

**GUI:** Datacenter → Summary (sức khoẻ toàn cụm) · Node → Summary (đồ thị CPU/RAM/IO/Net) ·
Node → Ceph → Dashboard. Đổi khung thời gian Hour / Day / Week ở góc phải mỗi đồ thị.

**CLI — cùng số liệu đó, lấy được bằng script:**

```bash
pvesh get /cluster/resources --type node --output-format yaml   # cpu/mem/uptime mọi node
pvesh get /nodes/pve1/status --output-format yaml | head -30    # loadavg, ksm, rootfs
pvesh get /nodes/pve1/rrddata --timeframe hour --output-format json | head -c 600; echo

pvesm status                      # storage nào active, dùng bao nhiêu %
ceph -s; ceph df; ceph osd perf   # sức khoẻ + latency commit/apply từng OSD
```

> `rrddata` là nguồn của mọi đồ thị trong GUI. Biết lấy bằng API nghĩa là tự động hoá được
> báo cáo, không phải chụp màn hình.

## Bước 2 — Cài & cấu hình công cụ giám sát: Metric Server

Proxmox đẩy metric ra ngoài theo chuẩn **Graphite** hoặc **InfluxDB**. Lab dùng Graphite/UDP vì
kiểm chứng được ngay mà không cần cài thêm phần mềm.

**Trên `jump` — mở cổng nghe để thấy metric chảy về:**

```bash
nc -u -l -p 2003 | head -20        # để chạy, sang bước kế
```

**Trên `pve1` — khai báo metric server** (hoặc GUI: Datacenter → Metric Server → Add → Graphite):

```bash
pvesh create /cluster/metrics/server/lab-graphite \
  --type graphite --server 10.0.10.1 --port 2003 --proto udp
pvesh get /cluster/metrics/server --output-format yaml
```

Quay lại cửa sổ `jump`: sau ~10 giây phải thấy các dòng dạng
`pve.nodes.pve1.cpu <giá trị> <timestamp>` — **đó là bằng chứng metric đang chảy**.

> Production thay `nc` bằng InfluxDB/Graphite thật rồi vẽ bằng Grafana. Cấu hình **phía Proxmox
> y hệt bước này** — chỉ đổi địa chỉ đích. Kiến trúc Prometheus + `pve-exporter` xem slide Bài 06.

**Dọn sau khi xác minh** (tránh node bắn UDP vào hư vô suốt phần còn lại của khoá):

```bash
pvesh delete /cluster/metrics/server/lab-graphite
```

## Bước 3 — Báo cáo hệ thống: `pvereport`

```bash
pvereport > /tmp/pvereport-$(hostname)-$(date +%F).txt
wc -l /tmp/pvereport-*.txt
grep -nE '^==|pve-manager|ceph version|Quorate|bond0' /tmp/pvereport-*.txt | head -20
```

`pvereport` gom phiên bản, storage, network, cluster, guest và log gần nhất vào **một file**.
Đây là thứ gửi kèm khi mở ticket hỗ trợ, và là ảnh chụp "hệ thống lúc còn tốt" để về sau so sánh.

## Bước 4 — Nhận diện bottleneck theo tầng

Chạy khi hệ thống đang bình thường để biết **hình dạng của trạng thái khoẻ**:

```bash
uptime                                   # loadavg so với số core (4)
vmstat 1 5                               # cột r (chờ CPU), si/so (swap = thiếu RAM)
iostat -x 1 3                            # %util, await — disk bận hay chậm
ss -s                                    # số kết nối
ceph osd perf                            # OSD nào latency lệch hẳn
```

| Triệu chứng | Tầng nghi trước | Lệnh xác nhận |
|---|---|---|
| loadavg cao, `r` lớn, %util thấp | CPU | `vmstat`, `top` |
| `si/so` khác 0 | RAM (swap) | `free -h`, `vmstat` |
| `await` cao, `%util` ~100% | Disk | `iostat -x`, `ceph osd perf` |
| Mạng chậm nhưng CPU/disk rảnh | Network / MTU | `ping -M do -s 8972`, `iperf3` |

## Bước 5 — Baseline hiệu năng với `fio`

Không có baseline thì "hệ thống chậm" là câu nói vô nghĩa. Đo **trước**, lưu lại, so sánh sau.

```bash
apt-get install -y fio
mkdir -p /mnt/pve/nfs-store/bench

# 4k random read — đo IOPS và latency (KHÔNG chạy lúc lớp đang tải nặng)
fio --name=r4k --filename=/mnt/pve/nfs-store/bench/t.img --size=512M \
    --rw=randread --bs=4k --iodepth=16 --numjobs=1 --runtime=30 --time_based \
    --direct=1 --group_reporting | tee /tmp/fio-4k-$(date +%F).txt

grep -E 'IOPS|lat .*avg' /tmp/fio-4k-*.txt
rm -f /mnt/pve/nfs-store/bench/t.img
```

Ghi lại **IOPS + latency trung bình** vào ghi chú của bạn.

> ⚠️ Con số trong lab nested **không** có ý nghĩa tuyệt đối — nhiều cụm dùng chung một máy vật lý.
> Thứ mang về production là **quy trình**: cùng một lệnh, cùng block size, cùng iodepth, chạy lúc hệ
> thống rảnh, lưu kết quả kèm ngày tháng. Baseline chỉ có giá trị khi lặp lại được y hệt.

## Bước 6 — Ngưỡng cảnh báo nên đặt

Không dựng hệ thống alert trong lab, nhưng chốt danh sách để mang về:

| Cảnh báo | Ngưỡng gợi ý | Vì sao |
|---|---|---|
| Node down / không quorate | ngay lập tức | mất quorum = pmxcfs read-only |
| Storage usage | **80%** (Ceph nearfull 85%) | Ceph đầy là **chặn ghi** toàn cụm |
| Backup job fail | mỗi lần fail | backup hỏng chỉ lộ ra lúc cần restore |
| Clock skew | > 50ms | Ceph báo lỗi, HA fence oan |
| OSD down / PG không `active+clean` | ngay lập tức | đang chạy giảm dự phòng |
| Disk SMART lỗi | ngay lập tức | thay trước khi hỏng hẳn |

> Triết lý: **cảnh báo phải hành động được**. Alert không ai xử lý thì sẽ bị ngó lơ, và alert thật
> chìm theo.

## Bước 7 — Xác nhận cuối (chạy trên pve1)

```bash
echo "== báo cáo =="
ls /tmp/pvereport-*.txt >/dev/null 2>&1 && echo "pvereport OK" || echo "pvereport FAIL"
echo "== baseline fio =="
ls /tmp/fio-4k-*.txt >/dev/null 2>&1 && echo "baseline OK" || echo "baseline FAIL"
echo "== metric server đã dọn =="
pvesh get /cluster/metrics/server --output-format json 2>/dev/null | grep -q 'lab-graphite' \
  && echo "metric server CHƯA dọn" || echo "metric server clean OK"
echo "== nền =="
pvecm status | grep -q 'Quorate: *Yes' && ceph health | grep -q HEALTH_OK && echo "cluster OK" || echo "cluster FAIL"
```

Kỳ vọng: tất cả `OK`. Xong bước này → capstone (`Lab06-Capstone.md`): instructor chạy `start.sh`, bạn nhận ticket sự cố.
