# Cách thay đổi cấu hình HAProxy nhanh chóng, không có Downtime
---
## Tổng quan

Chào các bạn, trong bài blog này mình sẽ hướng dẫn các bạn thay đổi cấu hình HAProxy ngay lập tức, không làm downtime dịch vụ thông qua HAProxy API.

HAProxy Runtime API được tích hợp vào HAProxy, cho phép thay đổi cấu hình HAProxy ngày lập tức, đồng thời HAProxy API cũng cho phép chiết xuất dữ liệu  thông kê về HAProxy. Không giống như các API HTTP thông thường, HAProxy API chạy thông qua TCP và Unix sockets. Đó là lý do khi cấu hình HAProxy API, ta phải thiết lập socket mới cho HAProxy (stats socket hoặc socket) và đôi khi trong một số tài liệu nói về HAProxy API lại sử dụng thuật ngữ 'socket' để thay thế. 

## Mô hình

### Cấu hình

```
CPU         2 core
RAM         4 GB
Disk        vda: os
Network     eth0: Network Public
```

### Mô hình triển khai

![](/images/haproxy-api/pic1.png)

## Cài đặt

### Bước 1: Cài đặt, cấu hình HTTPD

```
yum install httpd -y
cat /etc/httpd/conf/httpd.conf | grep 'Listen 80'
sed -i "s/Listen 80/Listen 10.10.11.89:8080/g" /etc/httpd/conf/httpd.conf

echo '<h1>Chào mừng tới Blog Cloud365</h1>' > /var/www/html/index.html

systemctl start httpd
systemctl enable httpd
```

### Bước 2: Cài đặt, cấu hình HAProxy

#### Cài đặt

```
sudo yum install wget socat -y
wget http://cbs.centos.org/kojifiles/packages/haproxy/1.8.1/5.el7/x86_64/haproxy18-1.8.1-5.el7.x86_64.rpm 
yum install haproxy18-1.8.1-5.el7.x86_64.rpm -y

echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf
```

#### Cấu hình HAProxy

```
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

echo '
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/run/haproxy.sock mode 600 expose-fd listeners level user
    stats socket ipv4@127.0.0.1:9999 level admin
    stats timeout 2m
    server-state-file /var/lib/haproxy/server-state

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option  forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    option                  http-server-close
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
    load-server-state-from-file global

listen stats
    bind :8000
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats admin if TRUE

listen web-backend
    bind 10.10.11.89:80
    balance leastconn
    cookie SERVERID insert indirect nocache
    mode  http
    option  forwardfor
    #http-request set-header REALCIP %[src]
    option  httpchk GET / HTTP/1.0
    option  httpclose
    option  httplog
    timeout  client 3h
    timeout  server 3h
    server node1 10.10.11.89:8080 weight 1 check cookie s1' > /etc/haproxy/haproxy.cfg
```

Lưu ý, để cấu hình HAProxy API cần quan tâm các cấu hình sau

```
global
    ...
    stats socket /var/run/haproxy.sock mode 600 expose-fd listeners level user
    stats socket ipv4@127.0.0.1:9999 level admin
    stats timeout 2m
    server-state-file /var/lib/haproxy/server-state
    ...
defaults
    load-server-state-from-file global
```

Giải thích cấu hình:
- `stats socket`: Cho phép kết nối tới socket HAProxy API
- `/var/run/haproxy.sock`: Đường dẫn tới file socket kết nối tới (Lưu ý: Bản HAProxy CE sẽ khác bản HAProxy EE, trong bài mình sử dụng bản HAProxy CE)
- `ipv4@127.0.0.1:9999`: Chỉ định địa chỉ và port được phép kết nối tới HAProxy API
- `server-state-file` và `load-server-state-from-file`: Lưu lại trạng thái server. Ví dụ, khi chúng ta chuyển trạng thái server thành `maintain` nhưng đây chỉ là trạng thái tạm thời. Đến khi khởi động lại HAProxy, trạng thái server đó sẽ trở lại trạng thái `active`. Để tránh tình trạng đó xảy ra, sử dụng 2 cấu hình `server-state-file` và `load-server-state-from-file`.

#### Cấu hình Log cho HAProxy

```
sed -i "s/#\$ModLoad imudp/\$ModLoad imudp/g" /etc/rsyslog.conf
sed -i "s/#\$UDPServerRun 514/\$UDPServerRun 514/g" /etc/rsyslog.conf
echo '$UDPServerAddress 127.0.0.1' >> /etc/rsyslog.conf

echo 'local2.*    /var/log/haproxy.log' > /etc/rsyslog.d/haproxy.conf

systemctl restart rsyslog
```

#### Khởi tạo dịch vụ HAProxy

```
systemctl restart haproxy
systemctl enable haproxy
```

### Kiểm tra

Truy cập Web, `http://10.10.11.89`

![](/images/haproxy-api/pic2.png)

Giao diện giám sát, `http://10.10.11.89:8000/stats`

![](/images/haproxy-api/pic3.png)

Truy cập HAProxy API
```
socat stdio /var/run/haproxy.sock
prompt

>
```

Kết quả

![](/images/haproxy-api/pic4.png)

## Sử dụng HAProxy API

### Truy cập HAProxy API
```
socat stdio /var/run/haproxy.sock
prompt

> help
```

Kết quả

```
  help           : this message
  prompt         : toggle interactive mode with prompt
  quit           : disconnect
  show tls-keys [id|*]: show tls keys references or dump tls ticket keys when id specified
  set ssl tls-key [id|keyfile] <tlskey>: set the next TLS key for the <id> or <keyfile> listener to <tlskey>
  show errors    : report last request and response errors for each proxy
  disable agent  : disable agent checks (use 'set server' instead)
  disable health : disable health checks (use 'set server' instead)
  disable server : disable a server for maintenance (use 'set server' instead)
  enable agent   : enable agent checks (use 'set server' instead)
  enable health  : enable health checks (use 'set server' instead)
  enable server  : enable a disabled server (use 'set server' instead)
  set maxconn server : change a server's maxconn setting
  set server     : change a server's state, weight or address
  get weight     : report a server's current weight
  set weight     : change a server's weight (deprecated)
  show sess [id] : report the list of current sessions or dump this session
  shutdown session : kill a specific session
  shutdown sessions server : kill sessions on a server
  clear table    : remove an entry from a table
  set table [id] : update or create a table entry's data
  show table [id]: report table usage stats or dump this table's contents
  clear counters : clear max statistics counters (add 'all' for all counters)
  show info      : report information about the running process
  show stat      : report counters for each proxy and server
  show schema json : report schema used for stats
  show startup-logs : report logs emitted during HAProxy startup
  show resolvers [id]: dumps counters from all resolvers section and
                     associated name servers
  set maxconn global : change the per-process maxconn setting
  set rate-limit : change a rate limiting value
  set severity-output [none|number|string] : set presence of severity level in feedback information
  set timeout    : change a timeout setting
  show env [var] : dump environment variables known to the process
  show cli sockets : dump list of cli sockets
  show fd [num] : dump list of file descriptors in use
  disable frontend : temporarily disable specific frontend
  enable frontend : re-enable specific frontend
  set maxconn frontend : change a frontend's maxconn setting
  show servers state [id]: dump volatile server information (for backend <id>)
  show backend   : list backends in the current running config
  shutdown frontend : stop a specific frontend
  set dynamic-cookie-key backend : change a backend secret key for dynamic cookies
  enable dynamic-cookie backend : enable dynamic cookies on a specific backend
  disable dynamic-cookie backend : disable dynamic cookies on a specific backend
  show cache     : show cache status
  add acl        : add acl entry
  clear acl <id> : clear the content of this acl
  del acl        : delete acl entry
  get acl        : report the patterns matching a sample for an ACL
  show acl [id]  : report available acls or dump an acl's contents
  add map        : add map entry
  clear map <id> : clear the content of this map
  del map        : delete map entry
  get map        : report the keys and values matching a sample for a map
  set map        : modify map entry
  show map [id]  : report available maps or dump a map's contents
  show pools     : report information about the memory pools usage
```

### Kiểm tra các backend server hiện có

```
> show servers state
```

Kết quả
```
1
# be_id be_name srv_id srv_name srv_addr srv_op_state srv_admin_state srv_uweight srv_iweight srv_time_since_last_change srv_check_status srv_check_result srv_check_health srv_check_state srv_agent_state bk_f_forced_id srv_f_forced_id srv_fqdn srv_port
3 web-backend 1 node1 10.10.11.89 2 0 1 1 209 15 3 4 6 0 0 0 - 8080
```

### Thay đổi trạng thái server

Cú pháp

```
disable server <backend>/<server>
enable health <backend>/<server>
```

Ví dụ
```
> disable server web-backend/node1
```

Kiểm tra

![](/images/haproxy-api/pic5.png)

Ví dụ
```
> enable server web-backend/node1
```

![](/images/haproxy-api/pic6.png)

### Bỏ qua health check

Cú pháp
```
disable health <backend>/<server> 
enable server <backend>/<server>
```

Ví dụ
```
> disable health web-backend/node1
```

Kết quả

![](/images/haproxy-api/pic7.png)

Ví dụ
```
> enable health web-backend/node1
```

![](/images/haproxy-api/pic6.png)

### Thay đổi cấu hình backend web

Cú pháp
```
set server <backend>/<server> agent [ up | down ]
set server <backend>/<server> health [ up | stopping | down ]
set server <backend>/<server> state [ ready | drain | maint ]
set server <backend>/<server> weight <weight>[%]
set server <backend>/<server> addr <ip> port <port number>
```

Ví dụ
```
> set server web-backend/node1 addr 10.10.11.88 port 80
IP changed from '10.10.11.89' to '10.10.11.88', port changed from '8080' to '80' by 'stats socket command'
```

Kiểm tra
```
> show servers state

1
# be_id be_name srv_id srv_name srv_addr srv_op_state srv_admin_state srv_uweight srv_iweight srv_time_since_last_change srv_check_status srv_check_result srv_check_health srv_check_state srv_agent_state bk_f_forced_id srv_f_forced_id srv_fqdn srv_port
3 web-backend 1 node1 10.10.11.88 0 0 1 1 6 7 0 0 7 0 0 0 - 80
```

### Lưu trạng thái các backend server

Mặc định có thể thay đổi trạng thái server trong backend như từ `ready` sang `maintain`. Nhưng khi restart trạng thái cấu hình sẽ mất vì vậy chúng ta cần lưu lại trạng thái lại hiện tại của các backend server

Giải thích các trạng thái của backend server
- ready: Backend server sẵn sàng phục vụ, được điều hướng request từ internet vào
- drain: Backend server sẵn sàng phục vụ, không được điều hướng request từ internet vào, tuy nhiên vẫn giữ các phiên hiện tại trên backend
- maint: Backend server không sẵn sàng phục vụ, không được điều hướng request từ internet vào

Lưu trạng thái

```
echo "show servers state" | socat stdio /var/run/haproxy.sock > /var/lib/haproxy/server-state
```

Kết quả
```
[root@ldap89 ~]# cat /var/lib/haproxy/server-state
1
# be_id be_name srv_id srv_name srv_addr srv_op_state srv_admin_state srv_uweight srv_iweight srv_time_since_last_change srv_check_status srv_check_result srv_check_health srv_check_state srv_agent_state bk_f_forced_id srv_f_forced_id srv_fqdn srv_port
3 web-backend 1 node1 10.10.11.89 2 0 1 1 311 15 3 4 6 0 0 0 - 8080
```

Lưu ý:
- Khi tồn tại file `server-state`, HAProxy sẽ tự động sử dụng các trạng thái trong file để áp dụng lên các backend server hiện tại. Nên lưu ý làm mới hoặc xóa file khi không cần thiết


## Tổng kết

Tới đây mình đã hướng dẫn các bạn sử dụng HAProxy API để thao tác, cấu hình HAProxy trực tiếp, không cần khởi động lại dịch vụ. Ngoài ra, HAProxy API còn hỗ trợ cấu hình trực tuyến các thành phần khác như ACL, Frontend, Session, v.v để biết thêm chi tiết, các bạn có thể tham khảo tại các link bên dưới. Cám ơn các bạn đã quan tâm

## Nguồn

http://haproxy.tech-notes.net/9-2-unix-socket-commands/

https://www.haproxy.com/blog/dynamic-configuration-haproxy-runtime-api/