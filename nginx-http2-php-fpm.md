
Mở file nginx.conf và điều chỉnh theo các hướng dẫn sau:

```
max_clients = worker_processes * worker_connections
```

Mặc định:

```
worker_processes = 1
worker_connections = 1024
```

worker_processes = số lượng cpu core trong server. để xem thông số cpu core ta dùng lệnh:

```
cat /proc/cpuinfo | grep processor
processor   : 0
processor   : 1
processor   : 2
processor   : 3
```

Như thông tin trên thì ta có 4 cpu core. Ta sẽ điều chỉnh thông số worker_processes = 4.
Không được điều chỉnh lớn hơn số lượng cpu core mà server có.

```
worker_connections = worker_processes * 1024
```

Xóa các thông tin về server nginx bằng config:

```
server_tokens off;
```

Thay đổi kích thước body của các http request và buffer:

```
client_max_body_size 20m;
client_body_buffer_size 128k;
```

Setup cache các file statics bằng config sau:

```
location ~* .(jpg|jpeg|gif|png|css|js|ico|xml)$ {  
   access_log    off;  
   log_not_found   off;  
   expires      360d;  
}
```

Thông thường liên lạc giữa Nginx vs PHP-FPM thông qua tcp socket.
Việc này gây ra chậm tốc độ so với sử dụng unix sockket.
Để thay đổi ta sửa config sau:

```
location ~* .php$ {
    fastcgi_index index.php;
    fastcgi_pass  unix:/var/run/php-fpm/php-fpm.sock;
    include fastcgi_params;
    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;  
    fastcgi_param  SCRIPT_NAME    $fastcgi_script_name;
}
```

Sample laravel

```
server {
    listen 80;
    server_name example.com;
    root /srv/example.com/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```
-----------------

Điều chỉnh PHP-FPM.

```
listen = /var/run/php-fpm/php-fpm.sock
user = ubuntu  
group = ubuntu  
request_slowlog_timeout = 5s  
slowlog = /var/log/php-fpm/slowlog-site.log  
listen.allowed_clients = 127.0.0.1  
pm = dynamic  
pm.max_children = 5  
pm.start_servers = 3  
pm.min_spare_servers = 2  
pm.max_spare_servers = 4  
pm.max_requests = 200  
listen.backlog = -1  
pm.status_path = /status  
request_terminate_timeout = 120s  
rlimit_files = 131072  
rlimit_core = unlimited  
catch_workers_output = yes  
env[HOSTNAME] = $HOSTNAME  
env[TMP] = /tmp  
env[TMPDIR] = /tmp  
env[TEMP] = /tmp  
```

Các thông tin lưu ý:

```
pm - chế độ quản lý proces của php-fpm gồm: static, ondemand, dynamic. Thông thường sử dụng dynamic.
pm.max_children = Số process con (child processes) tối đa được tạo (tương đương tổng số request có thể phục vụ)
pm.start_servers = Tổng số child processes được tạo khi khởi động php-fpm (được tính bằng công thức `min_spare_servers + (max_spare_servers – min_spare_servers) / 2` )
pm.min_spare_servers = Tổng số child process nhàn rỗi tối thiểu được duy trì.
pm.max_spare_servers = Tổng số child process nhàn rỗi tối đa được duy trì.
```

Cách tính `pm.max_children`:

B1. xác định số lượng bộ nhớ cần thiết trung bình của 1 process php-fpm.

```
ps -ylC php-fpm --sort:rss
```

Output sẽ tương tự như sau, column RSS sẽ chưa giá trị mà chúng ta cần xác định.

```
S   UID   PID  PPID  C PRI  NI   RSS    SZ WCHAN  TTY          TIME CMD
S     0 24439     1  0  80   0  6364 57236 -      ?        00:00:00 php-fpm
S    33 24701 24439  2  80   0 61588 63335 -      ?        00:04:07 php-fpm
S    33 25319 24439  2  80   0 61620 63314 -      ?        00:02:35 php-fpm
```

Như kết quả ở trên là khoảng 61620KB, tương đương với ~60MB/process

B2. Tính toán pm.max_children:

Nếu server có 4Gb chạy cả web và database. Chung ta để dành cho DB 1GB và 0,5GB cho buffer. Thì số lượng ram còn lại khoảng 2.5GB.

```
pm.max_children = 2560MB / 60Mb = 42.
```

Để tránh quá tải ta làm tròn xuống còn 40.

----------------
Cách tính `pm.min_spare_servers`

pm.min_spare_servers có giá trị tương đương với 20% của pm.max_children.
Nếu như với gía trị ở trên thì pm.min_spare_servers = 20% * 40 = 8
Cũng có hướng dẫn ghi là pm.min_spare_servers = (cpu cores) * 2

----------------
Cách tính pm.max_spare_servers

pm.max_spare_servers có giá trị tương đương với 60% của pm.max_children.
Nếu như với gía trị ở trên thì pm.max_spare_servers = 60% * 40 = 24
Cũng có hướng dẫn ghi là pm.max_spare_servers = (cpu cores) * 4

----------------
Cách tính pm.max_requests

Tham số này chính là số lượng request xử lý đồng thời mà server có thể chịu tải được, giá trị phụ thuộc vào pm.max_children và số lượng request trên 1s vào server.
Con số này tính toán cũng khá hên xui, nhưng có thể có 1 phương pháp là sử dụng tool ab của apache sau đó giảm giá trị dần dần sao cho phù hợp.

```
ab -n 5000 -c 100 https://codebase.vn
```

Khi chạy command trên thì có nghĩa là tạo ra 5000 request với 100 session hoạt động cùng lúc truy cập vào url https://codebase.vn.
Giá trị pm.max_requests có thể set là 1000 sau đó tăng/giảm dần đến mức phù hợp (Phù hợp là khi server vẫn còn chịu được tải).
