# Lab 01 — Challenge: "Node Người Cũ Để Lại"

> **Kịch bản:** Bạn tiếp quản hạ tầng từ một admin đã nghỉ việc. Node `pve3` được người đó "cài sẵn" — nhìn có vẻ chạy, nhưng chưa node nào join được cluster với nó. Nhiệm vụ: đưa `pve3` về trạng thái đạt chuẩn pre-flight, sẵn sàng join cluster ở bài sau.
>
> **Chấm:** `grade01.sh` từ jump host — kết quả **PASS/FAIL**, PASS khi **tất cả** tiêu chí đạt (cluster phải hoàn toàn khỏe mới sang được Bài 02).

## Acceptance criteria

Trên `pve3` (10.0.10.13):

1. `hostname --ip-address` trả về IP thật của node.
2. `/etc/hosts` phân giải đủ `pve1`, `pve2`, `pve3` — ping được cả 3 bằng hostname.
3. `apt update` chạy sạch — không 401, không repo lỗi; chỉ dùng repo no-subscription.
4. DNS hoạt động: `getent hosts download.proxmox.com` trả kết quả.
5. chrony chạy, enabled, đồng bộ nguồn `10.0.10.1`, offset < 0.1 giây.
6. Cả 3 disk dữ liệu (25 GB) **hoàn toàn trống**: không partition, không filesystem, không LVM.
7. Điều kiện nền vẫn nguyên vẹn: SSH root vào được, `pveproxy` và `pve-cluster` đang chạy.

## Cách tự kiểm

Chạy script sau từ jump host:

```bash
cd ~/graders && ./grade01.sh
```
