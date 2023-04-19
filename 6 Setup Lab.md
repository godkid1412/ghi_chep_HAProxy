# Trên 2 server chạy Wordpress

## Bước 1: Cài đặt LAMP + wordpress trên cả 2 server

### Cài đặt apache2:

```
sudo apt-get install apache2 apache2-utils
```

Bật apache2 web server chạy khi khởi động dùng:
```
sudo systemctl enable apache2
```

### Cài đặt Mysql

```
sudo apt-get install mysql-client mysql-server
```

### Cài đặt PHP

```
sudo apt-get install php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y
```
Sau khi cài đặt PHP cà các extension, ta cần khởi động lại Apache server:
```
sudo systemctl restart apache2
```

### Cài đặt Wordpress

```
wget -c http://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
```

Chuyển WordPress file sang thư mục gốc Apache
```
sudo mv wordpress/* /var/www/html/
```

Cài đặt quyền cho thư mục website sang chủ sở hữu là webserver:
```
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

### Cài đặt database cho wordpress

Để truy cập vào database server dùng:
```
sudo mysql -u root -p 
```

Tạo database, user, cấp quyền cho user dùng:
```
mysql> CREATE DATABASE wp_myblog;
mysql> CREATE USER 'good'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
mysql> GRANT ALL ON wp_myblog.* TO 'good'@'%';
mysql> FLUSH PRIVILEGES;
mysql> EXIT;
```

Chuyển sang thư mục /var/www/html, sửa tên file `wp-config-sample.php` thành `wp-config.php` và bỏ file `index.html` của apache2
```
cd /var/www/html/
sudo mv wp-config-sample.php wp-config.php
sudo rm -rf index.html
```

Sửa file `wp-config.php` thành như hình

![image](https://user-images.githubusercontent.com/54473576/231639006-0f609b99-67b5-4996-b451-638f3e287559.png)

Cài đặt wordpress, nhập `server_address/index.php` trên trình duyệt

![image](https://user-images.githubusercontent.com/54473576/231639771-2f1fab48-bc9f-429c-8f2f-7d5f19154b49.png)

![image](https://user-images.githubusercontent.com/54473576/231639816-84f6d609-d348-4957-ac55-3bfb7fcc15e0.png)

![image](https://user-images.githubusercontent.com/54473576/231639830-322b3e1d-6e1b-46f7-930a-2d4d9d8a3f8a.png)

## Bước 2: cài đặt group replication

Bước này dùng khi 1 trong 2 database server được thay đổi thì nó đồng bộ sang server còn lại

### Tạo UUID để tìm nhóm ở Mysql\

Tại máy chủ WP1 dùng:
```
wp@good:~$ uuidgen
90a22fa4-80ed-4e0e-8218-988721c98443
```
Lưu mã đó ra chỗ khác rồi dùng sau

### Cài đặt Group Replication trong mysql configuration file

```
sudo vim /etc/mysql/my.cnf
```

Thêm đoạn sau để cài đặt trên 2 server:
```
. . .

[mysqld]

# General replication settings
disabled_storage_engines="MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"
gtid_mode = ON
enforce_gtid_consistency = ON
master_info_repository = TABLE
relay_log_info_repository = TABLE
binlog_checksum = NONE
log_slave_updates = ON
log_bin = binlog
binlog_format = ROW
transaction_write_set_extraction = XXHASH64
loose-group_replication_bootstrap_group = OFF
loose-group_replication_start_on_boot = OFF
loose-group_replication_ssl_mode = REQUIRED
loose-group_replication_recovery_use_ssl = 1
# Shared replication group configuration
loose-group_replication_group_name = ""
loose-group_replication_ip_whitelist = ""
loose-group_replication_group_seeds = ""

# Single or Multi-primary mode? Uncomment these two lines
# for multi-primary mode, where any host can accept writes
#loose-group_replication_single_primary_mode = OFF
#loose-group_replication_enforce_update_everywhere_checks = ON

# Host specific replication configuration
server_id = 
bind-address = ""
report_host = ""
loose-group_replication_local_address = ""
```

#### Chia sẻ cài đặt group replication trên 2 máy:

loose-group_replication_group_name điền mã vừa sinh ra ở câu lênh `uuidgen`
```
# Shared replication group configuration
 loose-group_replication_group_name = "90a22fa4-80ed-4e0e-8218-988721c98443"
 loose-group_replication_ip_whitelist = "10.0.0.10,10.0.0.20"
 loose-group_replication_group_seeds = "10.0.0.10:33061,10.0.0.20:33061"
```

#### Chọn  cài đặt 1 nhóm chính hay nhóm nhiều chính

```
. . .
# Single or Multi-primary mode? Uncomment these two lines
# for multi-primary mode, where any host can accept writes
#loose-group_replication_single_primary_mode = OFF
#loose-group_replication_enforce_update_everywhere_checks = ON
. . .
```

Nếu cài đặt nhóm 1 chính thì để nguyên, còn nhóm đa chính thì bỏ comment 2 dòng cuối.

Cài đặt phải giống nhau ở từng server


#### Cài đặt cấu hình cho từng server

```
# Host specific replication configuration
server_id = 
bind-address = ""
report_host = ""
loose-group_replication_local_address = ""
```

Ở server 1:
```
# Host specific replication configuration
 server_id = 1
 bind-address = "10.0.0.10"
 report_host = "10.0.0.10"
 loose-group_replication_local_address = "10.0.0.10:33061"
```

Ở server 2
```
# # # Host specific replication configuration
 server_id = 2
 bind-address = "10.0.0.20"
 report_host = "10.0.0.20"
 loose-group_replication_local_address = "10.0.0.20:33061"
```

#### Khởi động lại mysql

```
sudo systemctl restart mysql
```

#### Cài đặt Replication User và bật Group Replication Plugin

Vào mysql trên 2 server dùng:
```
sudo mysql -u root -p
```
Khi bật group replication thì sẽ truyền cho các máy khác qua `bin log` tạo nên xung đột sau khi tạo xong nên cần tắt khi cài đặt.

```
mysql> SET SQL_LOG_BIN=0;
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY 'password' REQUIRE SSL;
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
mysql> FLUSH PRIVILEGES;
mysql> SET SQL_LOG_BIN=1;
```

Tiếp theo, đặt `group_replication_recovery` kênh để sử dụng người dùng sao chép mới của bạn và mật khẩu được liên kết của họ. Sau đó, mỗi máy chủ sẽ sử dụng các thông tin đăng nhập này để xác thực với nhóm:
```
CHANGE REPLICATION SOURCE TO SOURCE_USER='repl', SOURCE_PASSWORD='password' FOR CHANNEL 'group_replication_recovery';
```

Nếu sử dụng bản mysql cũ hơn 8.0.23, ta cần sử dụng cú pháp kế thừa của Mysql:
```
CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='password' FOR CHANNEL 'group_replication_recovery';
```

Cài đặt plugin `group_replication` để khởi tạo nhóm
```
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
```
`group_replication` có thể đổi tên khác theo ý muốn.

```
mysql> SHOW PLUGINS;
+----------------------------+----------+--------------------+----------------------+---------+
| Name                       | Status   | Type               | Library              | License |
+----------------------------+----------+--------------------+----------------------+---------+
|                            |          |                    |                      |         |
| . . .                      | . . .    | . . .              | . . .                | . . .   |
|                            |          |                    |                      |         |
| group_replication          | ACTIVE   | GROUP REPLICATION  | group_replication.so | GPL     |
+----------------------------+----------+--------------------+----------------------+---------+
45 rows in set (0.00 sec)
```

#### Start group_replication

Nếu được đặt biến `group_replication_bootstrap_group` sẽ cho một thành viên biết rằng họ không nên mong đợi nhận thông tin từ các máy chủ và thay vào đó nên thành lập một nhóm mới và tự bầu chọn thành viên chính. Bạn có thể bật biến này bằng lệnh sau:
```
SET GLOBAL group_replication_bootstrap_group=ON;
```

Dùng máy chủ WP1 bắt đầu group_replication
```
mysql> SET GLOBAL group_replication_bootstrap_group=ON;
```

Sau đó, bạn có thể đặt `group_replication_bootstrap_group` trở lại OFF, vì trường hợp duy nhất thích hợp là khi không có thành viên nhóm hiện tại:
```
mysql> SET GLOBAL group_replication_bootstrap_group=OFF;
```

Xem các thành viên trong nhóm:
```
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+---------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST   | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+---------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | 13324ab7-1b01-11e7-9dd1-22b78adaa992 | 10.0.0.10     |        3306 | ONLINE       | PRIMARY     | 8.0.28         | XCom                       |
+---------------------------+--------------------------------------+---------------+-------------+--------------+-------------+----------------+----------------------------+
1 row in set (0.00 sec)
```

Khởi động group_replication trên máy chủ WP2:
```
mysql> START GROUP_REPLICATION;
```

Kiểm tra các thành viên trong nhóm:
```
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE | MEMBER_ROLE | MEMBER_VERSION | MEMBER_COMMUNICATION_STACK |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+
| group_replication_applier | 066f21f0-ced0-11ed-8e31-000c29c07655 | 10.0.0.10   |        3306 | ONLINE       | PRIMARY     | 8.0.32         | XCom                       |
| group_replication_applier | ceff2b7e-d200-11ed-b800-000c2965d9cd | 10.0.0.20   |        3306 | ONLINE       | PRIMARY     | 8.0.32         | XCom                       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+-------------+----------------+----------------------------+
```

#### Tự động tham gia nhóm khi MySQL khởi động

```
sudo vim /etc/mysql/my.cnf
```

Đặt `loose-group_replication_start_on_boot` thành `ON`
```
[mysqld]
. . .
loose-group_replication_start_on_boot = ON
. . .
```


### Cài đặt đồng bộ thư mục upload ảnh

#### Dùng NFS Server

Dùng WP1 làm NFS server, WP2 làm NFS client

##### Tại máy chủ WP1

Cài đặt NFS server:
```
apt install nfs-kernel-server
```

Để thiết lập chia sẻ thư mục sửa file `/etc/export`

Thêm dòng bên dưới để chia sẻ và phân quyền cho thư mục
`
/var/www/html/wp-content/uploads 10.0.0.20(rw,sync,no_subtree_check,no_root_squash)
`

Trong đó
- /var/www/html/wp-content/uploads: Thư mục được chia sẻ
- 10.0.0.20: địa chỉ có thể truy cập, có thể dùng dải mạng
- (rw,sync,no_subtree_check,no_root_squash): quyền truy cập

Các quyền trong NFS:
- ro: Read only.
- rw: Read – write.
- noaccess: Denied access.
- root_squash: Ngăn remote root users.
- no_root_squash: Cho phép remote root users.
- no_subtree_check: Không kiểm tra subtree

##### Tại máy chủ WP2

Mount thư mục được chia sẻ đến thư mục upload ảnh của wordpress:

```
sudo mount 10.0.0.10:/var/www/html/wp-content/uploads /var/www/html/wp-content/uploads
```

#### Dùng crontab và rsync

##### Tại máy chủ WP1

```
 sudo ssh-keygen -t rsa -b 2048
```
Nhấn Enter cho đến khi được như hình:
![image](https://www.linuxshelltips.com/wp-content/uploads/2021/10/Generate-SSH-Key-in-Linux.png)

Copy private key và public key sang server còn lại dùng:
```
sudo ssh-copy-id -i /root/.ssh/id_rsa.pub wp@10.0.0.20
```

Cấu hình cron job để tự động đồng bộ trên server
```
sudo crontab -e
```
Để đồng bộ mỗi 30s ghi vào file như sau:
```
* * * * * sudo rsync -avzhe ssh wp@10.0.0.20:/var/www/html/wp-content/ /var/www/html/wp-content/
* * * * * sleep 30; sudo rsync -avzhe ssh wp@10.0.0.20:/var/www/html/wp-content/ /var/www/html/wp-content/
```

##### Tại máy chủ WP2

```
 sudo ssh-keygen -t rsa -b 2048
```
Nhấn Enter cho đến khi được như hình:
![image](https://www.linuxshelltips.com/wp-content/uploads/2021/10/Generate-SSH-Key-in-Linux.png)

Copy private key và public key sang server còn lại dùng:
```
sudo ssh-copy-id -i /root/.ssh/id_rsa.pub wp@10.0.0.10
```

Cấu hình cron job để tự động đồng bộ trên server
```
sudo crontab -e
```
Để đồng bộ mỗi 30s ghi vào file như sau:
```
* * * * * sudo rsync -avzhe ssh wp@10.0.0.10:/var/www/html/wp-content/ /var/www/html/wp-content/
* * * * * sleep 30; sudo rsync -avzhe ssh wp@10.0.0.10:/var/www/html/wp-content/ /var/www/html/wp-content/
```

# Trên các server HAProxy

## Cấu hình HAProxy

Dùng lệnh sau để cài đặt haproxy trên cả 2 server:
```
sudo apt-get install haproxy -y
```

Để sửa file cấu hình HAProxy trên server dùng lệnh:
```
sudo vim /etc/haproxy/haproxy.cfg
```

Sau khi mở file như:
```
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http
```
### Trên server HAProxy1 

Viết cấu hình cho HAProxy để cân bằng tải cho 2 sever wordpress:
```
frontend www
        bind 10.0.0.4:80
        mode tcp
        option tcplog
        default_backend wp_server
backend wp_server
        balance roundrobin
        mode tcp
        option tcplog
        server wp1 10.0.0.10:80 check weight 1
        server wp2 10.0.0.20:80 check weight 1
listen stats
        bind :8000
        stats refresh 1s
        stats enable
        stats uri /monitoring
        stats auth good:password

```

Đoạn cấu hình sẽ redirect các traffic từ 10.0.0.4:80 sang 10.0.0.10:80 hoặc 10.0.0.20:80

Cấu hình backend server là round robin, các server backend có trọng số 1:1

Ngoài ra còn cấu hình thêm stat để xem các trạng thái và thông số của từng server

### Trên server HAProxy2

Viết cấu hình cho HAProxy để cân bằng tải cho 2 sever wordpress:
```
frontend www
        bind 10.0.0.4:80
        mode tcp
        option tcplog
        default_backend wp_server
backend wp_server
        balance roundrobin
        mode tcp
        option tcplog
        server wp1 10.0.0.10:80 check weight 2
        server wp2 10.0.0.20:80 check weight 1
listen stats
        bind :8000
        stats refresh 1s
        stats enable
        stats uri /monitoring
        stats auth good:password

```

Đoạn cấu hình sẽ redirect các traffic từ 10.0.0.4:80 sang 10.0.0.10:80 hoặc 10.0.0.20:80

Cấu hình backend server là round robin, các server backend có trọng số 2:1

Ngoài ra còn cấu hình thêm stat để xem các trạng thái và thông số của từng server


## Cấu hình Keepalived trên cả 2 server

Dùng lệnh sau để cài đặt haproxy trên cả 2 server:
```
sudo apt-get install keepalived -y
```

Để sửa file cấu hình HAProxy trên server dùng lệnh:
```
sudo vim /etc/keepalived/keepalived.conf
```

Ở đây, ta sử dụng server HAProxy 1 làm server chính, HAProxy 2 làm server phụ trong trường hợp HAProxy 1 bị DOWN thì nó lên làm thay nhiệm vụ

#### Trên server HAProxy 1:

```
global_defs {
   smtp_server localhost
   smtp_connect_timeout 30
}

vrrp_script chk_haproxy{
        script "killall -0 haproxy"
        interval 2
        weigh 4
}

vrrp_instance VI_1 {
        state MASTER
        interface ens35
        virtual_router_id 101
        priority 101
        advert_int 1
        authentication {
        auth_type PASS
        auth_pass 1111
        }

        virtual_ipaddress {
                10.0.0.4
        }

        track_script{
                chk_haproxy
        }
}
```

#### Trên server HAProxy 2
```
global_defs {
   smtp_server localhost
   smtp_connect_timeout 30
}

vrrp_script chk_haproxy{
        script "killall -0 haproxy"
        interval 2
        weigh 4
}

vrrp_instance VI_1 {
        state BACKUP
        interface ens35
        virtual_router_id 101
        priority 100
        advert_int 1
        authentication {
        auth_type PASS
        auth_pass 1111
        }

        virtual_ipaddress {
                10.0.0.4
        }

        track_script{
                chk_haproxy
        }
}
```

Ở cấu hình trên được cài đặt script để kiểm tra xem service `haproxy` có chạy hay không, nếu service `haproxy` ở server HAProxy 1 không chạy thì nó sẽ sang server 2

Khởi động lại dịch vụ `keepalived` trên cả 2 server:
```
sudo systemctl restart keepalived
```

