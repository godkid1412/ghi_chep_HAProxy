![Screenshot 2023-04-12 231050](https://user-images.githubusercontent.com/54473576/231518164-2c8bb510-8d8e-44e7-a35e-3e62e2105824.png)

Trong bài lab gồm 4 server:
- Server HAProxy 1 và 2 làm nhiệm vụ cân bằng tải. Cả 2 đều dùng thuật toàn round robin, điểm khác biệt là HaProxy 1 có trọng số (weight) là 1:1, HAProxy 2 có trọng số là 2:1 (2 request vào WP1 và 1 request và WP2)
- Server HAProxy1 và 2 đều được cấu hình HAProxy và Keepalived dùng cho trường hợp server HAProxy 1 bị down thì có server HAProxy 2 đứng ra làm nhiệm vụ thay nó
- Server WP1 và WP2 được cài đặt làm web server
- WP1 và WP2 được cài đặt group replication theo kiểu đa chính (Multi-Primary Environment)
- Ngoài ra 2 server đc cài đặt tự động để đồng bộ 1 thư mục upload ảnh.

