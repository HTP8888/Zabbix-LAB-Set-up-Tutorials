# 🚀 Zabbix Lab (Zabbix 6.0 LTS trên Ubuntu 20.04)

Hướng dẫn cài đặt và cấu hình **Zabbix Server 6.0 LTS** trên Ubuntu 20.04 (phù hợp cho môi trường học tập / lab / production nhỏ).

---

## 📌 Yêu cầu hệ thống

- Ubuntu Server 20.04 LTS (hoặc Desktop có cài LAMP).
- PHP 7.4 (mặc định).
- MySQL hoặc MariaDB.
- Apache2.
- RAM tối thiểu 2 GB, user có quyền `sudo`.

---

## 🔧 Các bước cài đặt

### Bước 1 – Cập nhật hệ thống & cài các gói phụ trợ  
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y vim wget php-cgi php-common php-mbstring \
php-net-socket php-gd php-xml-util php-mysql php-bcmath php-imap \
php-snmp libapache2-mod-php libc6
```

### Bước 2 – Cấu hình PHP để phù hợp Zabbix  
```bash
sudo a2enconf php7.*-cgi
sudo nano /etc/php/7.4/apache2/php.ini
```
Chỉnh các thông số:
```
max_execution_time = 300
max_input_time    = 300
memory_limit      = 200M
post_max_size     = 32M
upload_max_filesize = 16M
date.timezone     = Asia/Ho_Chi_Minh
```
Khi setup bạn sử dụng lệnh Ctrl+W để tìm dòng lệnh, lệnh Ctrl+O để lưu và lệnh Crtl+X để thoát.

Khởi động lại Apache:
```bash
sudo systemctl restart apache2
```

### Bước 3 – Tạo database & user cho Zabbix  
```bash
sudo mysql -uroot -p
```
Trong MySQL shell:
```sql
DROP DATABASE IF EXISTS zabbixdb;
CREATE DATABASE zabbixdb CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER IF NOT EXISTS 'zabbixuser'@'localhost' IDENTIFIED WITH mysql_native_password BY 'zabbixPWD';
GRANT ALL PRIVILEGES ON zabbixdb.* TO 'zabbixuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Bước 4 – Thêm repository Zabbix chính thức  
```bash
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu20.04_all.deb
sudo dpkg -i zabbix-release_latest+ubuntu20.04_all.deb
sudo apt update
```

### Bước 5 – Cài đặt Zabbix server + frontend + agent  
```bash
sudo apt install -y zabbix-server-mysql zabbix-frontend-php \
zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

### Bước 6 – Import schema Zabbix vào DB  
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbixuser -p zabbixdb
```

### Bước 7 – Cấu hình Zabbix server  
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
Chỉnh:
```
DBHost=localhost
DBName=zabbixdb
DBUser=zabbixuser
DBPassword=zabbixPWD
```

### Bước 8 – Khởi động Zabbix và Apache  
```bash
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

### Bước 9 – Cấu hình Firewall (nếu dùng UFW)  
```bash
sudo ufw enable
sudo ufw allow 10050
sudo ufw allow 10051
sudo ufw reload
sudo ufw status
```
Có thể bạn chỉ cần dùng lệnh sau để tắt hẳn tường lửalửa
```bash
sudo ufw disable 
```
### Bước 10 – Hoàn tất qua giao diện Web UI  
Mở browser:
```
http://<IP-server>/zabbix
```
Thực hiện các bước:
1. Chọn ngôn ngữ → **Next step**  
2. Kiểm tra yêu cầu → **Next**  
3. Điền DB:
   - Database name: `zabbixdb`  
   - User: `zabbixuser`  
   - Password: `zabbixPWD` → **Next**  
4. Đặt tên Zabbix server → **Next**  
5. Xác nhận → **Next**, rồi **Finish**  
6. Đăng nhập với:  
   - **User**: `Admin`
   - **Password**: `zabbix`

---

## ⚠️ Các lỗi thường gặp khi cài đặt

1. **Lỗi MySQL `command not found`**  
   - Chưa cài MySQL server → cài bằng:  
     ```bash
     sudo apt install mysql-server -y
     ```

2. **Lỗi `Access denied for user 'zabbixuser'@'localhost'`**  
   - Sai mật khẩu hoặc chưa cấp quyền cho user.  
   - Fix: chạy lại trong MySQL:  
     ```sql
     ALTER USER 'zabbixuser'@'localhost' IDENTIFIED WITH mysql_native_password BY 'zabbixPWD';
     FLUSH PRIVILEGES;
     ```

3. **Lỗi khi import schema: `Table 'role' already exists`**  
   - Bạn đã import schema một phần rồi.  
   - Fix: Drop DB và tạo lại sạch sẽ:
     ```sql
     DROP DATABASE zabbixdb;
     CREATE DATABASE zabbixdb CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
     ```

4. **Lỗi `SUPER privilege required and binary logging enabled`**  
   - Khi import schema mà MySQL bật binlog.  
   - Fix:
     ```sql
     SET GLOBAL log_bin_trust_function_creators = 1;
     ```
     hoặc import bằng tài khoản `root`.

5. **Lỗi Web UI: `dbversion not found`**  
   - Do chưa import schema thành công.  
   - Fix: chạy lại lệnh import schema:
     ```bash
     zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbixuser -p zabbixdb
     ```

6. **Lỗi mất DB sau khi reboot**  
   - Do `datadir` của MySQL bị trỏ sai (ví dụ `/tmp`).  
   - Fix: trong `/etc/mysql/mysql.conf.d/mysqld.cnf` bật lại:
     ```
     datadir = /var/lib/mysql
     ```

---

## 📦 Backup & Restore trong DB Zabbix  

Backup:
```bash
mysqldump -uzabbixuser -p zabbixdb > zabbixdb_backup.sql
```
Restore:
```bash
mysql -uzabbixuser -p zabbixdb < zabbixdb_backup.sql
```
Chúc các bạn thành công <3
---

## 👨‍💻 Tác giả  

- Hoàng Trần Phong – PTIT  
- GitHub: [HTP8888](https://github.com/HTP8888)
