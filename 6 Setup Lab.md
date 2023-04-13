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

