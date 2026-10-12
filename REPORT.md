# Báo cáo kỹ thuật: Phân tích kiến trúc Container vs Virtual Machine

## 1. Bối cảnh

Trong quá trình vận hành hệ thống VPS truyền thống, doanh nghiệp phải chịu nhiều
hạn chế: mỗi dịch vụ cần một máy ảo riêng với hệ điều hành đầy đủ, thời gian khởi
động chậm, tốn tài nguyên và khó tự động hóa. Việc chuyển dịch sang Docker dựa
trên ảo hóa mức hệ điều hành (Operating-system-level virtualization) giúp giải
quyết các vấn đề trên. Báo cáo này phân tích sự khác biệt kiến trúc giữa ảo hóa
phần cứng (VM/Hypervisor) và ảo hóa hệ điều hành (Container Engine), cơ chế cô lập
kernel qua Namespaces, quản lý tài nguyên qua cgroups, cùng các thông số thực tế
trích xuất từ `docker info`.

## 2. So sánh VM vs Container theo 5 tiêu chí

| Tiêu chí | Virtual Machine (Hypervisor) | Container (Docker Engine) |
|---|---|---|
| **Dung lượng Image** | Lớn, thường 1–20 GB vì chứa toàn bộ Guest OS (kernel, driver, thư viện hệ thống). | Nhỏ, thường 5–500 MB vì chia sẻ kernel với host, chỉ đóng gói ứng dụng và thư viện phụ thuộc. |
| **Thời gian khởi động** | Chậm, từ vài chục giây đến vài phút do phải boot Guest OS và khởi tạo kernel. | Rất nhanh, thường dưới 1 giây vì chỉ tiến trình ứng dụng được khởi tạo, không cần boot OS. |
| **Hao phí OS** | Cao: mỗi VM chạy một kernel và toàn bộ dịch vụ hệ thống, tiêu tốn CPU/RAM nền đáng kể. | Thấp: không nhân bản kernel, nhiều container chia sẻ chung nhân Linux của host. |
| **Mức độ cô lập** | Cô lập mạnh ở mức phần cứng (ring-based), ranh giới bảo mật chắc chắn nhất. | Cô lập ở mức tiến trình qua Namespaces; nhẹ hơn, cần cấu hình bảo mật bổ sung (seccomp, AppArmor). |
| **Khả năng mở rộng** | Co giãn chậm, tốn tài nguyên; thường phù hợp scale theo chiều dọc. | Co giãn nhanh, nhẹ; rất phù hợp kiến trúc microservices, scale theo chiều ngang. |

**Kết luận:** VM phù hợp khi cần cô lập tuyệt đối và chạy nhiều hệ điều hành khác
nhau; Container vượt trội về mật độ triển khai, tốc độ và tính linh hoạt — là lý do
chính doanh nghiệp chuyển từ VPS sang Docker.

## 3. Cơ chế cô lập và quản lý tài nguyên trong Container

### 3.1 Namespaces — cô lập không gian tài nguyên

Namespace là tính năng của kernel Linux cho phép một tiến trình chỉ "nhìn thấy"
một tập hợp tài nguyên riêng, tạo cảm giác như đang chạy trên một máy độc lập:

- **PID namespace** — cô lập Process: container có cây tiến trình riêng, tiến trình
  đầu tiên được gán PID 1. Container này không nhìn thấy tiến trình của host hay
  container khác.
- **Network namespace** — cô lập Network: mỗi container có bảng route, card mạng ảo
  (veth), địa chỉ IP và cổng riêng, tách biệt hoàn toàn với mạng host.
- **Mount namespace** — cô lập Mount points: container có hệ thống tệp riêng biệt,
  chỉ thấy filesystem của image và các volume được gắn tường minh.
- Ngoài ra còn có UTS (hostname), IPC, User namespaces góp phần cô lập toàn diện.

### 3.2 Control Groups (cgroups) — giới hạn tài nguyên

cgroups là cơ chế kernel dùng để đo lường, kiểm soát và giới hạn tài nguyên mà
một nhóm tiến trình được phép sử dụng:

- **Giới hạn CPU:** qua tham số `--cpus` hoặc `--cpu-shares`, đặt trần và trọng số
  chia sẻ CPU cho container, tránh một container chiếm dụng toàn bộ CPU.
- **Giới hạn RAM:** qua `--memory` và `--memory-swap`, giới hạn bộ nhớ tối đa; khi
  vượt ngưỡng, tiến trình bị OOM-killer xử lý thay vì làm sập cả host.
- Ngoài ra cgroups còn kiểm soát I/O đĩa (`--device-read-bps`) và số lượng PID.

Nhờ Namespaces (ranh giới hiển thị) kết hợp cgroups (ranh giới tài nguyên),
container đạt được mức cô lập và kiểm soát đủ tốt với chi phí rất thấp.

## 4. Thông số trích xuất từ `docker info`

Kết quả thực thi trên hệ thống (Windows + Docker Desktop, engine Linux/WSL2):

| Thông số | Giá trị |
|---|---|
| **Server Version** | 29.8.0 (Docker Desktop 4.91.0) |
| **Cgroup Driver** | cgroupfs |
| **Cgroup Version** | 2 |
| **Số lượng Containers** | 9 |
| **Số lượng Images** | 10 |
| Storage Driver | overlayfs |
| Kernel Version | 6.18.33.2-microsoft-standard-WSL2 |
| CPUs / Total Memory | 22 / 15.34 GiB |
| Docker Root Dir | /var/lib/docker |

## 5. Kiểm tra

Cả hai lệnh đều thực thi thành công bằng user thường, **không cần `sudo`** (trên
Windows không tồn tại cơ chế `sudo` của Linux; Docker Desktop cấp quyền qua
user group `docker-users`):

```text
$ docker version
Client:
 Version:           29.8.0
 API version:       1.56
 OS/Arch:           windows/amd64
 Context:           desktop-linux

Server: Docker Desktop 4.91.0 (239619)
 Engine:
  Version:          29.8.0
  API version:      1.56 (minimum version 1.40)
  OS/Arch:          linux/amd64
 containerd:
  Version:          v2.3.4
 runc:
  Version:          1.4.3

$ docker info
 Server Version: 29.8.0
 Storage Driver: overlayfs
 Cgroup Driver: cgroupfs
 Cgroup Version: 2
 Kernel Version: 6.18.33.2-microsoft-standard-WSL2
 Operating System: Docker Desktop
 CPUs: 22
 Total Memory: 15.34GiB
 Containers: 9
 Images: 10
 Docker Root Dir: /var/lib/docker
```

Kiểm tra nhanh Cgroup Driver bằng lệnh gợi ý (dùng `Select-String` tương đương
`grep` trên PowerShell):

```text
$ docker info | Select-String -Pattern "cgroup"
 Cgroup Driver: cgroupfs
 Cgroup Version: 2
```

## 6. Kết luận

Container không thay thế hoàn toàn VM về khả năng cô lập bảo mật, nhưng nhờ chia sẻ
kernel, kết hợp Namespaces và cgroups, Docker mang lại mật độ triển khai cao, khởi
động tức thời và tiết kiệm tài nguyên — đáp ứng tốt nhu cầu microservices và CI/CD.
Đó là cơ sở kỹ thuật để doanh nghiệp chuyển từ VPS sang Docker.
