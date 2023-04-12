# Agent check

`Agent check` là 1 nơi HAProxy kết nối với 1 ctrinh đang chạy ở backend server

```
backend servers
   server server1 192.168.0.10:80 check  weight 100  agent-check  agent-addr 192.168.0.10  agent-port 8080  agent-inter 5s  agent-send ping\n
```
Haproxy gửi 1 agent check mỗi 5s và lắng nghe ở 192.168.0.10 tại port 8080

# ARP - Bản trả phí

HAProxy ALOHA gửi ARP request đến toàn thiết bị ở mạng để xác nhận máy chủ nhận được IP đã có. Nếu đúng với server mà ko trả lời thì server đó đánh fails

Các option ở ARP check

|Option|Description|
|---|---|
|`timeout <seconds>`|Sau khoảng thời gian mà không có phản hồi thì đánh fails, mặc định 1 nửa của `interval`|
|`interval <seconds>`|Khoảng cách giữa 2 lần ktra, mặc định 10s|
|`source <ip>`|srcIP sử dụng để kiểm tra|
|`port <port>`|Cổng sử dụng, mặc định cổng thực máy chủ, nếu có|
|`rise <count>`|Máy chủ coi là hoạt động sau `count` lần, mặc định 1|
|`fall <count>`|Máy chủ coi là ko hoạt động sau `count` lần, mặc định 1|
|`inhibit`|Nếu mấy chủ không hoạt động, trọng số (weight) đặt là 0,|

# HTTP

LB gửi cho các máy chủ 1 yêu cầu HTTP. Nếu fails thì health check fails.

Để bật active health check, chọn LB layer 7, và thêm option `httpchk` ở phía backend. Thêm tham số `check` ở sau mỗi server

`option httpchk` chỉ định gửi HTTP request đến server và đợi trả về (successful response). Response status trong khoảng 2xx, 3xx

## Bật `active health check`

```
backend be_myapp
  option httpchk
  server srv1 10.0.0.1:80   check
  server srv2 10.0.0.2:8080 check
```


## `Response state` là 200 thì successful health check

```
backend be_myapp
  option httpchk
  http-check expect status 200
  server srv1 10.0.0.1:80   check
  server srv2 10.0.0.2:8080 check
```

## Coi tất cả trạng thái là hợp lệ ngoại trừ 5xx

```
backend be_myapp
  option httpchk
  http-check expect ! rstatus ^5
  server srv1 10.0.0.1:80   check
  server srv2 10.0.0.2:8080 check
```
## Thay đổi ngưỡng (threshold) của fails health check

Ngưỡng mặc định là 2
```
backend be_myapp
  option httpchk
  server srv1 10.0.0.1:80   check fall 5 rise 10
  server srv2 10.0.0.2:8080 check fall 5 rise 10
```

## Thay đổi tần suất của health check

```
backend be_myapp
  option httpchk
  server srv1 10.0.0.1:80   check inter 4s fastinter 1s downinter 1m
  server srv2 10.0.0.2:8080 check inter 4s fastinter 1s downinter 1m
```

Health check mỗi 4s. 

Khi thất bại trong health check đầu tiên thì tần suất kiểm tra mỗi 1s

Khi server đamg ngừng hoạt động thì kiểm tra mỗi 1 phút

# ICMP

Khi LB dịch vụ dựa trên UDP, 

HAproxy gửi các gói ping đến máy chủ. Máy chủ ko phản hồi thì đánh fails

```
director syslog_servers 10.0.0.9:514 UDP
  balance roundrobin                      # load balancing algorithm
  mode gateway                            # forwarding mode

  option icmpcheck
  check interval 5s timeout 1s            # advanced check parameters
  server syslog1 10.0.0.1:514 check
  server syslog2 10.0.0.2:514 check
```


Cách sử dụng:
- Thêm một tham số `check`vào mỗi dòng mà bạn muốn kích hoạt kiểm tra tình trạng.
- Thêm `option icmpcheck` vào cùng một phần. Nó không có đối số.
- Thêm dòng checkvào phần để định cấu hình các tùy chọn khác. Nó sử dụng cú pháp sau:
```
check { [timeout <seconds>] [interval <seconds>] [source <ip>] [port <port>] [rise <count>] [fall <count>] [inhibit] }
```
Các option của icmp check
|Option|Description|
|---|---|
|`timeout <seconds>`|Sau khoảng thời gian mà không có phản hồi thì đánh fails, mặc định 1 nửa của `interval`|
|`interval <seconds>`|Khoảng cách giữa 2 lần ktra, mặc định 10s|
|`source <ip>`|srcIP sử dụng để kiểm tra|
|`port <port>`|Cổng sử dụng, mặc định cổng thực máy chủ, nếu có|
|`rise <count>`|Máy chủ coi là hoạt động sau `count` lần, mặc định 1|
|`fall <count>`|Máy chủ coi là ko hoạt động sau `count` lần, mặc định 1|
|`inhibit`|Nếu mấy chủ không hoạt động, trọng số (weight) đặt là 0,|

# TCP 

## Active health checks

### Check TCP port

Server HAProxy kết nối với cổng TCP của máy chủ bên backend. Kiểm tra thành công nếu máy chủ trả về gói SYN/ACK. Hoạt động bằng cách cho tham số `check` vào mỗi dòng server bên backend

```
backend be_myapp
   server srv1 10.0.0.1:80 check
   server srv2 10.0.0.2:80 check
```
Load Balancer kết nối với máy chủ qua cổng 80

```
backend be_myapp
   server srv1 10.0.0.1:80 check port 8080
   server srv2 10.0.0.2:80 check port 8080
```
Để kiểm tra tình trạng đến một cổng khác với cổng mà lưu lượng truy cập bình thường được gửi đến, dùng tham số `port`.

```
backend be_myapp
   server srv1 10.0.0.1:80 check fall 5
   server srv2 10.0.0.2:80 check fall 5
```
5 lần liên tiếp kiểm tra fails đưa máy chủ thành trạng thái `DOWN`

```
backend be_myapp
  option httpchk
  server srv1 10.0.0.1:80   check inter 4s fastinter 1s downinter 1m
  server srv2 10.0.0.2:8080 check inter 4s fastinter 1s downinter 1m
```
Health check mỗi 4s. 

Khi thất bại trong health check đầu tiên thì tần suất kiểm tra mỗi 1s

Khi server đamg ngừng hoạt động thì kiểm tra mỗi 1 phút


