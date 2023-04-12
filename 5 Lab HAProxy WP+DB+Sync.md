![Screenshot 2023-04-12 231050](https://user-images.githubusercontent.com/54473576/231518164-2c8bb510-8d8e-44e7-a35e-3e62e2105824.png)

Trong bài lab gồm 4 server:
- Server HAProxy 1 và 2 làm nhiệm vụ cân bằng tải. Cả 2 đều dùng thuật toàn round robin, điểm khác biệt là HaProxy 1 có trọng số (weight) là 1:1, HAProxy 2 có trọng số là 2:1 (2 request vào WP1 và 1 request và WP2)
- Server HAProxy1 và 2 đều được cấu hình HAProxy và Keepalived dùng cho trường hợp server HAProxy 1 bị down thì có server HAProxy 2 đứng ra làm nhiệm vụ thay nó
- Server WP1 và WP2 được cài đặt làm web server
- WP1 và WP2 được cài đặt group replication theo kiểu đa chính (Multi-Primary Environment)
- Ngoài ra 2 server đc cài đặt tự động để đồng bộ 1 thư mục upload ảnh.

## Server HAProxy1 chạy bình thường:

HAProxy 1 sẽ chia tải 1:1 với từng server bên phía backend

![image](https://user-images.githubusercontent.com/54473576/231528417-460d0ad6-7657-40ed-a022-6d9f6d3a1296.png)


## Server Haproxy 1 bị down service haproxy

Khi đó sẻver HAProxy 2 đứng ra làm thay nhiệm vụ của HAProxy 1

HAProxy 2 sẽ chia tải 2:1 như đã cấu hình cho từng server bên phía backend

![image](https://user-images.githubusercontent.com/54473576/231530879-de0ab1d1-65a8-4818-9e04-678983fe77ba.png)

## Chức năng đồng bộ trên Mysql

### Ban đầu cả 2 có chung các bài post

![image](https://user-images.githubusercontent.com/54473576/231532007-7e243a29-147e-44d9-ab5c-304c59160ef0.png)


### Xóa 1 bài post bên phía server WP1 và kết quả sau khi làm mới trang tại server WP2

![image](https://user-images.githubusercontent.com/54473576/231532396-26b5704b-b3b2-43a5-b225-5bc1c149aba5.png)

![image](https://user-images.githubusercontent.com/54473576/231532455-22108266-8f61-49d8-b0b0-d31d7a623429.png)

### Thêm 1 bài post bên phía server WP2 và kết quả sau khi làm mới trang tại server WP1

![image](https://user-images.githubusercontent.com/54473576/231532923-e835ba20-fd49-491e-aa5a-60a8c94b8d46.png)

![image](https://user-images.githubusercontent.com/54473576/231532983-f9d5380b-ae10-413c-8784-90734578327c.png)

## Chức năng đồng bộ thư mục upload ảnh

### Khi chưa đồng bộ thư mục upload ảnh

Nguyên nhân: khi upload ảnh với tại mỗi server sẽ lưu tại trên local. Trước đó ta có đồng bộ database, khi upload ở server WP1 thì dữ liệu để hiển thị ảnh dược truyền sang DB2. Trong khi đó trên server WP2 không có hình ảnh đó để hiển thị

![image](https://user-images.githubusercontent.com/54473576/231534400-f9b2fec7-810f-4ecf-93d7-c7f42754b0fe.png)

### Khi thư mục upload được đồng bộ

![image](https://user-images.githubusercontent.com/54473576/231535779-3484bf93-b63d-4b0e-ae91-95454d8d663b.png)
