# Tổng quan và các khái niệm quan trọng về cân bằng tải trong HAProxy

---

## Tổng quan
HAProxy viết tắt của High Availability Proxy, là công cụ mã nguồn mở nổi tiếng ứng dụng cho giải pháp cân bằng tải TCP/HTTP cũng như giải pháp máy chủ Proxy (Proxy Server). HAProxy có thể chạy trên các mỗi trường Linux, Solaris, FreeBSD. Công dụng phổ biến nhất của HAProxy là cải thiện hiệu năng, tăng độ tin cậy của hệ thống máy chủ bằng cách phân phối khối lượng công việc trên nhiều máy chủ (như Web, App, cơ sở dữ liệu). HAProxy hiện đã và đang được sử dụng bởi nhiều website lớn như GoDaddy, GitHub, Bitbucket, Stack Overflow, Reddit, Speedtest.net, Twitter và trong nhiều sản phẩm cung cấp bởi Amazon Web Service.

## Thuật ngữ trong HAProxy
Có rất nhiều thuật ngữ và khái niệm được sử dụng trong HAProxy khi nối về cân bằng tải (Load balancing) và máy chủ. Ở đây, tôi sẽ tập trung vào các khái niệm thông dụng được sử dụng nhiều trong HAProxy

### Access Control List (ACL)
Access Control List (ACL) sử dụng để kiểm tra một số điều kiện và thực hiện hành động tiếp theo dựa trên kết quả kiểm tra(VD lựa chọn một server, chặn một request). Sử dụng ACL cho phép điều tiết lưu lượng mạng linh hoạt dựa trên các yếu tố khác nhau (VD: dựa theo đường dẫn, dựa theo số lượng kết nối tới backend)

### Backend
Backend là tập các server nhận các request đã được điều tiết (HAProxy điều tiết các request tới các backend). Các Backend được định nghĩa trong mục `backend` khi cấu hình HAProxy. 

2 cấu hình thường được định nghĩa trong mục `backend`:
- Thuật toán cân bằng tải (Round Robin, Least Connection, IP Hash)
- Danh sách các Server, Port (Nhận, xử lý request)

Backend có thể chứa một hoặc nhiều server. Việc thêm nhiều server vào backend sẽ cải thiện tải, hiệu năng, tăng độ tin cậy dịch vụ. Và khi một server thuộc backend không khả dụ, các server khác thuộc backend sẽ chịu tải thay cho server xảy ra vấn đề.

Ví dụ minh họa
```
backend web-backend
   balance roundrobin
   server web1 web1.yourdomain.com:80 check
   server web2 web2.yourdomain.com:80 check

backend blog-backend
   balance roundrobin
   mode http
   server blog1 blog1.yourdomain.com:80 check
   server blog1 blog1.yourdomain.com:80 check
```

`balance roundrobin` chỉ định thuật toán cân bằng tải: các Request phân phối tuần tự tới các server, đây cũng là phương thức được sử dụng mặc định.

`mode http` chỉ định proxy layer 7 sẽ được sử dụng

### Frontend
Frontend định nghĩa cách các request điều tiết tới backend. Các cấu hình Frontend được định nghĩa trong mục `frontend` khi cấu hình HAProxy. 

Các cấu hình frontend bao gồm các thành phần:
- Tập các IP và port (VD: 10.10.10.86:80, *:443)
- Các ACL
- Các backend nhận, xử lý request.

## Các loại cân bằng tải

### Không có cân bằng tải

Kiến trúc đơn giản nhất khi triển khai ứng dụng Web

![](/images/img-tongquan-haproxy/pic1.png)

Trong ví dụ, người dùng sẽ kết nối trực tiếp tới Webserver (https://blog.cloud365.vn/), và không sử dụng dịch vụ cân bằng tải. Nếu web server xảy ra vấn đề, người dùng sẽ không thể kết nối tới Web được nữa. Và nếu trong trường hợp nhiều người cùng truy cập, webserver có thể không đáp ứng được các request, dẫn đến trải nhiệm sử dụng sẽ giảm xuống.

### Layer 4 Load Balancing
Cách đơn giản nhất để cân bằng tải các request tới nhiều server là sử dụng cân bằng tải mức layer 4 TCP (Tầng giao vận - transport layer). Phương pháp sẽ điều hướng các request dựa trên IP và Port. Theo ví dụ, nếu request tới địa chỉ `https://blog.cloud365.vn/` thì request sẽ được điều hướng tới backend `web-backend` để xử lý.

Lưu ý: 
- Hai máy chủ web cần phục vụ nội dung giống nhau. Nếu không, người dùng sẽ nhận thông tin không thống nhất (Tùy theo thuật toán cân bằng tải).
- Nên sử dụng chung database giữ 2 web server.

![](/images/img-tongquan-haproxy/pic2.png)

### Layer 7 Load Balancing
Phương pháp phức tạp hơn, cân bằng tải sử dụng tại tầng layer 7 mức request (Tầng ứng dụng - Application layer). Sử dụng bộ cần bằng tại layer 7 sẽ điều hướng request tới các backend khác nhau dựa trên nội dung của request.

Chế độ này cho phép bạn có thể triển khai nhiều web server khác nhau trên cùng 1 domain.

![](/images/img-tongquan-haproxy/pic3.png)

Trong hình, nếu người dùng gửi request tới 'https://blog.cloud365.vn/', haproxy sẽ điều hướng request tới 1, còn khi người dùng request tới `https://blog.cloud365.vn/about/` haproxy sẽ điều hường request tới `web-2-backend`

## Các thuật toán cân bằng tải
Thuật toán cân bằng tải được sử dụng nhắm định nghĩa các request được điều hướng tới các server nằm trong backend trong quá trình load balancing. HAProxy cung cấp một số thuật toán mặc định:
- `roundrobin`: các request sẽ được chuyển đến server theo lượt. Đây là thuật toán mặc định được sử dụng cho HAProxy
- `leastconn`: các request sẽ được chuyển đến server nào có ít kết nối đến nó nhất
- `source`: các request được chuyển đến server bằng các hash của IP người dùng. Phương pháp này giúp người dùng đảm bảo luôn kết nối tới một server

## Sticky Sessions
Một số ứng dụng yêu cầu người dùng phải giữ kết nối tới cùng một server thuộc backend, để giữ kết nối giữa client với một backend server bạn có thể sử dụng tùy chọn `sticky sessions`, xem thêm tại link dưới.

[Xem thêm](https://www.haproxy.com/blog/load-balancing-affinity-persistence-sticky-sessions-what-you-need-to-know/)

## Health Check
HAProxy sử dụng `health check` để phát hiện các backend server sẵn sàng xử lý request. Kỹ thuật này sẽ tránh việc loại bỏ server khỏi backend thủ công khi backend server không sẵn sàng. `health check` sẽ cố gắnh thiết lập kết nối TCP tới server để kiểm tra backend server có sẵn sàng xử lý request.

Nếu `health check` không thể kết nối tới server, nó sẽ tự động loại bỏ server khởi backend, các traffic tới sẽ không được forward tới server cho đến khi nó có thể thực hiện được `health check`. Nếu tất cả server thuộc backend đều xảy vấn đề, dịch vụ sẽ trở trên không khả dụ (trả lại status code 500) cho đến khi 1 server thuộc backend từ trạng thái không khả dụ chuyển sang trạng thái sẵn sàng.

## Nguồn

https://www.digitalocean.com/community/tutorials/an-introduction-to-haproxy-and-load-balancing-concepts

https://viblo.asia/p/huong-dan-su-dung-haproxy-cho-load-balancing-ung-dung-4P856jp95Y3

https://www.haproxy.com/blog/load-balancing-affinity-persistence-sticky-sessions-what-you-need-to-know/