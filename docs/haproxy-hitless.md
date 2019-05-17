# HAProxy Hitless - Thay đổi cấu hình HAProxy không có downtime
---
## Tổng quan

Chào các bạn, trong bài blog mình sẽ hướng dẫn các bạn thay đổi cấu hình, khởi động lại HAProxy không có downtime.

Đầu tiên chúng ta cần làm rõ, thế nào là `tải liên tục`. `Tải liên tục` hay `tải liền mạch` liên quan tới cập nhật cấu hình liên tục HAProxy mà ko ảnh hưởng tới trải nhiệm người dùng hay gây downtime cho hệ thống. Hoạt động quản trị dịch vụ thông thường yêu cầu dừng hoặc chạy lại 1 số dịch vụ, dẫn đến hiện tượng trì hoãn hoặc ko ổn định trong dịch vụ. Vì vậy, người quản trị thường tránh việc khởi động lại và phải lên lịch xử lý khi cần thay đổi cấu hình dịch vụ.

Quay lại quá khứ. Vào Năm 2006, haproxy 1.2.11 release tính năng soft reload, cơ chế đơn giản:
- Bước 1: Tiến trình mới cố năng bind tất cảc các port.
- Bước 2: Nếu nó lỗi, gửi tín hiệu tới tiến trình cũ, báo tiến trình cũ thả các port đang sử dụng.
- Bước 3: Tiến trình mới thử lại, cố gắng bind các port.
- Bước 4: Nếu tiến trình mới vẫn không thể bind các port, tiến trình mới sẽ từ bỏ, gửi tín hiệu tới tiến trình cũ báo tiến trình cũ tiếp tục xử lý.
- Bước 5: Nếu tiến trình mới xử lý thành công, nó gửi tín hiệu tới tiến trình cũ báo tiến trình mới đã nhận port và sẽ tiếp xử lý các kết nối sắp tới.

Giải pháp trên tưởng chừng ổn những thực sự không ổn. Vấn đề tại bước 2 và 3 khi PORT không được bind bởi tiến cũ hoặc tiến trình mới nó sẽ xảy ra downtime dịch vụ người dùng (khi tải cao sẽ là 1 thảm họa). Ngày nay, các dịch vụ Micro Service hay các hệ thống lớn thường xuyên yêu cầu khởi động lại dịch vụ, cập nhật cấu hình và không chấp nhận bất kỳ 1 kết nối nào xảy ra vấn đề do hệ thống Load balancer (LB). Để giải quyết vấn đề này, sau 7 năm liên tục thử nghiệm tìm kiếm giải pháp tới phiên bản 1.8, HAProxy đã công bố 2 giải pháp ổn định cho phép khởi động haproxy không có downtime.

2 giải pháp:
- Sử dụng IPTable dành cho các phiên bản nhỏ hơn 1.8
- Giải pháp `Hitless Reloads` được tích hợp bên trong HAProxy phiển bản 1.8

Trong bài mình sẽ sử dụng giải pháp thứ 2 đã có sẵn trong HAProxy. Để xem chi tiết hơn về các giải pháp trên, các bạn tham khảo link bên dưới

Lưu ý:
- Giải pháp `Hitless Reloads` được tích hợp bắt đầu từ phiên bản `HAProxy Enterprise 1.7r1` và `HAProxy 1.8` trở lên.

## Mô hình

### Mô hình

![](/images/haproxy-api/pic1.png)

### Cấu hình

```
CPU         2 core
RAM         4 GB
Disk        vda: os
Network     eth0: Network Public
```

### Chuẩn bị
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
init 6
```

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
Lưu ý, để cấu hình HAProxy Hitless reload đơn giản bổ sung cấu hình sau:
- `stats socket /var/run/haproxy.sock mode 600 expose-fd listeners level user`: Cấu hình này sẽ cho phép tiến trình mới của HAProxy khi khởi động lại có thể tái sử dụng các socket của tiến trình cũ.

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

pic 1

Câu lệnh restart haproxy không có downtime

```
haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -sf $(cat /run/haproxy.pid) -x /var/run/haproxy.sock
```

Để kiểm tra HAProxy reload có downtime hay không, mình sẽ sử dụng tool apache bench để thử nghiệm. 

Trong bài mình sẽ sử dụng 1 client có cài đặt sẵn apache tool và cùng dải với node HAProxy. Kịch bản như sau, mình sẽ tạo tổng 15000 request, với 500 request đồng thời tới HAProxy, trong thời gian thực hiện test mình sẽ reload hay restart dịch vụ HAProxy và kiểm chứng

#### Kiểm chứng khả năng reload không downtime của HAProxy
Bài test:
```
ab -r -c 500 -n 15000 http://10.10.11.89/
```

Bên dưới 2 bash shell thực hiện liên tục restart haproxy với dạng thông thường và dạng `hitless` môi 3s.

Restart thông thường, lưu ý: chạy trước khi thực hiện bài test, sau khi có kết quả thì các bạn `CTRL + C` để tắt script khởi động lại HAProxy
```
while true
do
    sleep 3
    systemctl restart haproxy
done
```

Restart không downtime, lưu ý: chạy trước khi thực hiện bài test, sau khi có kết quả thì các bạn `CTRL + C` để tắt script khởi động lại HAProxy
```
while true
do
    sleep 3
    haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -sf $(cat /run/haproxy.pid) -x /var/run/haproxy.sock
done
```

#### Kết quả

Với restart HAProxy thông thường kết quả:

```
[root@client ~]# ab -r -c 500 -n 15000 http://30.0.0.10/

Server Software:        Apache/2.4.6
Server Hostname:        30.0.0.10
Server Port:            80

Document Path:          /
Document Length:        42 bytes

Concurrency Level:      500
Time taken for tests:   3.016 seconds
Complete requests:      15000
Failed requests:        1703
   (Connect: 0, Receive: 433, Length: 844, Exceptions: 426)
Write errors:           0
Total transferred:      5083081 bytes
HTML transferred:       594678 bytes
Requests per second:    4972.71 [#/sec] (mean)
Time per request:       100.549 [ms] (mean)
Time per request:       0.201 [ms] (mean, across all concurrent requests)
Transfer rate:          1645.62 [Kbytes/sec] received
```

- Lưu ý: Giá trị `Failed requests = 1703` tức trong quá trình xử lý 15000 request đã có 1703 request xử lý thất bại => có downtime

Với restart HAProxy không có downtime
```
[root@client ~]# ab -r -c 500 -n 15000 http://30.0.0.10/

Server Software:        Apache/2.4.6
Server Hostname:        30.0.0.10
Server Port:            80

Document Path:          /
Document Length:        42 bytes

Concurrency Level:      500
Time taken for tests:   6.780 seconds
Complete requests:      15000
Failed requests:        0
Write errors:           0
Total transferred:      5385000 bytes
HTML transferred:       630000 bytes
Requests per second:    2212.51 [#/sec] (mean)
Time per request:       225.988 [ms] (mean)
Time per request:       0.452 [ms] (mean, across all concurrent requests)
Transfer rate:          775.67 [Kbytes/sec] received
```

- Lưu ý: Giá trị `Failed requests = 0` tức trong quá trình xử lý 15000 request đã có 0 request xử lý thất bại => Không có downtime


## Nguồn

https://www.haproxy.com/blog/hitless-reloads-with-haproxy-howto/

https://www.haproxy.com/blog/truly-seamless-reloads-with-haproxy-no-more-hacks/
---

Thực hiện bởi <a href="https://cloud365.vn/" target="_blank">cloud365.vn</a>
