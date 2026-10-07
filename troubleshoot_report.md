# Bài 5: Chẩn đoán và xử lý xung đột cổng 8082

## Hiện tượng

Khi `spring-app.service` khởi động, ứng dụng báo lỗi `Address already in use` vì
cổng TCP `8082` đang được một tiến trình khác sử dụng. Quy trình dưới đây ưu tiên
gửi tín hiệu `SIGTERM` để tiến trình có cơ hội kết thúc an toàn; chỉ dùng
`SIGKILL` khi tiến trình không phản hồi.

> Các PID trong phần kết quả minh họa phụ thuộc vào từng máy. Khi thực hành trên
> máy Linux, cần lưu lại đầu ra thực tế để thay cho các PID mẫu `3271` và `2146`.

## 1. Tạo tình huống xung đột

Nếu dịch vụ đang chạy từ bài 3, dừng dịch vụ trước khi tạo tiến trình giả lập:

```bash
sudo systemctl stop spring-app.service
python3 -m http.server 8082 > /dev/null 2>&1 &
SIMULATOR_PID=$!
echo "Simulator PID: ${SIMULATOR_PID}"
```

Kết quả minh họa:

```text
[1] 3271
Simulator PID: 3271
```

## 2. Chẩn đoán tiến trình chiếm cổng

Kiểm tra socket TCP đang lắng nghe trên cổng `8082`:

```bash
sudo ss -tlnp | grep ':8082'
```

Kết quả trước khi xử lý:

```text
LISTEN 0      5            0.0.0.0:8082       0.0.0.0:*    users:(("python3",pid=3271,fd=3))
```

Lấy PID một cách trực tiếp bằng `lsof`, sau đó dùng `ps` để xác minh đầy đủ user,
PID, PPID, trạng thái và câu lệnh đã khởi chạy tiến trình:

```bash
PORT_PID=$(sudo lsof -t -iTCP:8082 -sTCP:LISTEN)
echo "PID giữ cổng 8082: ${PORT_PID}"
ps -p "${PORT_PID}" -o user,pid,ppid,stat,lstart,cmd
```

Thông tin tiến trình ghi nhận trong lần chạy minh họa:

```text
PID giữ cổng 8082: 3271
USER       PID  PPID STAT                  STARTED CMD
student   3271  3198 S    Wed Oct  7 22:30:14 2026 python3 -m http.server 8082
```

Như vậy PID `3271`, tiến trình `python3 -m http.server 8082`, chính là tiến trình
giả lập đang chiếm cổng; không kết thúc nhầm tiến trình khác chỉ dựa vào tên.

## 3. Giải phóng cổng an toàn

Gửi `SIGTERM` trước và chờ tối đa 5 giây:

```bash
sudo kill "${PORT_PID}"

for attempt in 1 2 3 4 5; do
  if ! sudo kill -0 "${PORT_PID}" 2>/dev/null; then
    break
  fi
  sleep 1
done

if sudo kill -0 "${PORT_PID}" 2>/dev/null; then
  echo "Tiến trình không phản hồi SIGTERM; gửi SIGKILL"
  sudo kill -9 "${PORT_PID}"
fi
```

Xác nhận không còn tiến trình lắng nghe trên cổng trước khi khởi động ứng dụng:

```bash
if sudo ss -tlnp | grep -q ':8082'; then
  echo "Cổng 8082 vẫn đang bị chiếm"
else
  echo "Cổng 8082 đã được giải phóng"
fi
```

Kết quả:

```text
Cổng 8082 đã được giải phóng
```

## 4. Khởi động lại Spring Boot và xác minh

```bash
sudo systemctl restart spring-app.service
sudo systemctl status spring-app.service --no-pager
sudo ss -tlnp | grep ':8082'
ps -o user,pid,cmd -C java
```

Kết quả mong đợi từ dịch vụ đã cấu hình ở bài 3:

```text
● spring-app.service - Spring Boot Application
     Loaded: loaded (/etc/systemd/system/spring-app.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-10-07 22:32:03 +07
   Main PID: 2146 (java)
     CGroup: /system.slice/spring-app.service
             └─2146 /usr/bin/java -jar /opt/spring-app/app.jar

LISTEN 0      100                *:8082             *:*    users:(("java",pid=2146,fd=12))

USER             PID CMD
spring-runner    2146 /usr/bin/java -jar /opt/spring-app/app.jar
```

## Kết luận

Trước khi xử lý, cổng `8082` thuộc về tiến trình giả lập `python3` (PID mẫu
`3271`). Sau khi xác minh bằng `ss`, `lsof` và `ps`, tiến trình được kết thúc bằng
`SIGTERM`; `SIGKILL` chỉ là phương án dự phòng. Cuối cùng,
`spring-app.service` hoạt động ở trạng thái `active (running)` và cổng `8082`
được chuyển sang tiến trình `java` chạy `/opt/spring-app/app.jar` dưới user
`spring-runner`.

Nếu dịch vụ không khởi động thành công, xem 100 dòng log gần nhất bằng:

```bash
sudo journalctl -u spring-app.service -n 100 --no-pager
```
