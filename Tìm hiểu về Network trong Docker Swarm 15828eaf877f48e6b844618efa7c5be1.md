# Tìm hiểu về Network trong Docker Swarm

# Chuẩn bị

Chuẩn bị ít nhất 03 server cài đặt sẵn Docker. Có thể sử dụng [Ansible playbook](https://github.com/thuongnn/example-ansible-project.git) để cài đặt Docker đồng thời cho các server hoặc cài đặt Docker trên từng server với command sau:

```bash
➜  curl -sSL https://get.docker.io | bash
```

**Chú ý**: Docker Swarm cần mở một số protocols và ports dưới đây để giao tiếp giữa các server:

- **TCP port 2377** for cluster management communications
- **TCP** and **UDP port 7946** for communication among nodes
- **UDP port 4789** for overlay network traffic

Kiểm tra tường lửa của server xem có bị chặn các protocols và ports trên hay không bằng command:

```bash
➜  iptables-save
```

Nếu bị chặn, hãy mở protocols và ports trên bằng command:

```bash
iptables -I INPUT -p tcp -m tcp --dport 2377 -j ACCEPT
iptables -I INPUT -p tcp -m tcp --dport 7946 -j ACCEPT
iptables -I INPUT -p udp -m udp --dport 7946 -j ACCEPT
iptables -I INPUT -p udp -m udp --dport 4789 -j ACCEPT
```

Tìm hiểu thêm về iptables command [ở đây](https://wiki.matbao.net/iptables-la-gi-cach-cau-hinh-bao-mat-ubuntu-vps-linux-firewall/).

# Thiết lập cụm Docker Swarm

Kiến trúc triển khai sẽ bao gồm 01 manager node và 02 worker node:

![Ti%CC%80m%20hie%CC%82%CC%89u%20ve%CC%82%CC%80%20Network%20trong%20Docker%20Swarm%2015828eaf877f48e6b844618efa7c5be1/docker-swarm-architecture.png](Ti%CC%80m%20hie%CC%82%CC%89u%20ve%CC%82%CC%80%20Network%20trong%20Docker%20Swarm%2015828eaf877f48e6b844618efa7c5be1/docker-swarm-architecture.png)

1. SSH vào `manager` node và chạy command dưới đây để khởi tạo swarm cluster:

```bash
➜  docker swarm init
```

2. Sau khi init sẽ có một ghi chú được in ra màn hình bao gồm token dùng để join các work node vào cụm. SSH lần lượt vào `worker-1` và `worker-2` thực hiện command như hướng dẫn ở ghi chú trên để join vào cụm:

```bash
➜  docker swarm join --token <TOKEN> \
  --advertise-addr <IP-ADDRESS-OF-WORKER-1> \
  <IP-ADDRESS-OF-MANAGER>:2377
```

3. Sau khi join lần lượt  `worker-1` và `worker-2`, có thể kiểm tra danh sách các node khả dụng trên `manager` node bằng command:

```bash
➜  docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
3ahbvxay68yn43lojz7j8uqdg *   server-1   Ready     Active         Leader           20.10.7
xjlbrbhzfr2b2do6eosod7rt2     server-2   Ready     Active                          20.10.7
1860sgfoof3ujljs8f1zfk7h3     server-3   Ready     Active                          20.10.7
```

# Tìm hiểu về Docker Swarm Network

## Triển khai Nginx service với Docker Swarm

Trên `manager` node, tạo overlay network `nginx-net` và triển khai Nginx service kết nối tới `nginx-net`. Nginx service sẽ publish port `8080` ra bên ngoài map với port `80` bên trong Nginx container:

```bash
➜  docker network create -d overlay nginx-net
➜  docker service create \
  --name my-nginx \
  --publish target=80,published=8080 \
  --replicas=2 \
  --network nginx-net \
  thuongnn1997/nginx:debug
```

Kiểm tra Nginx service đang chạy trên server nào bằng command:

```bash
➜  docker service ps my-nginx
ID             NAME         IMAGE                      NODE       DESIRED STATE   CURRENT STATE            ERROR     PORTS
opdskafvlrcl   my-nginx.1   thuongnn1997/nginx:debug   server-1   Running         Running 15 minutes ago             
6gtue1osr897   my-nginx.2   thuongnn1997/nginx:debug   server-2   Running         Running 15 minutes ago
```

Để phân biệt nội dung trả về từ Nginx service, sử dụng `docker exec` vào từng Nginx container và sửa nội dung `index.html` như sau:

```bash
# docker exec -it my-nginx.1.opdskafvlrclhc38t0tcw5uup /bin/bash
➜  echo "Welcome to nginx-1 from server-1" > /usr/share/nginx/html/index.html
```

```bash
# docker exec -it my-nginx.2.6gtue1osr897u5i8fafdtoc3u /bin/bash
➜  echo "Welcome to nginx-2 from server-2" > /usr/share/nginx/html/index.html
```

## Kiểm tra

Thực hiện curl command nhiều lần vào một địa chỉ IP bất kỳ trong 03 server với port `8080` như ví dụ dưới đây:

```bash
➜  curl 171.244.143.222:8080
Welcome to nginx-2 from server-2
➜  curl 171.244.143.222:8080
Welcome to nginx-1 from server-1
➜  curl 171.244.143.222:8080
Welcome to nginx-2 from server-2
```

- Nhận thấy rằng truy cập vào một địa chỉ IP nhiều lần thì lại nhận được các kết quả khác nhau cho mỗi lần truy cập (kết quả phản hồi từ Nginx service lần lượt của container `nginx-1` và `nginx-2`)
- Dù có truy cập vào bất cứ server nào với port `8080` (kể cả Nginx service không chạy trên server đó) thì đều nhận được phản hồi từ Nginx service.

---

Liệt kê tất cả các loại Docker network trên server bằng command dưới đây:

```bash
➜  docker network ls
NETWORK ID     NAME              DRIVER    SCOPE
10864fa65d57   bridge            bridge    local
404e32f8d992   docker_gwbridge   bridge    local
4efb973544ed   host              host      local
rasrehzw9jfe   ingress           overlay   swarm
jtlbkwkmjz9w   nginx-net         overlay   swarm
fe2c98e1bb90   none              null      local
```

Theo lý thuyết, đối với Docker Swarm Network sẽ có các loại network có vai trò khác nhau dưới đây:

- **`nginx-net`** (overlay network) là private network do người dùng tạo ra để cho các service trên mạng này có thể giao tiếp nội bộ với nhau, đảm bảo chỉ những service join cùng private network mới có thể giao tiếp với nhau.
- **`ingress`** (overlay network) là overlay network mặc định được quản lý bởi Docker, khi một service published port ra bên ngoài thì sẽ được join vào network này.
- **`docker_gwbridge`** (bridge network) cũng là network mặc định được quản lý bởi Docker, nó hoạt động tương tự như `docker0`. Network này giúp các service bên trong container có thể giao tiếp với máy host bên ngoài.

## Ingress Network hoạt động như thế nào?

Như chúng ta đã biết thì `ingress` là một loại network đặc biệt đóng vai trò như Load Balancer giúp cho các request từ client được forward đến replica container nằm bất cứ đâu trên các node. Khi triển khai dịch vụ với tham số `--publish target=80,published=8080`, các replica container sẽ được tự động join vào `ingress network`. Vậy chúng hoạt động như thế nào?

Chúng ta đã biết 02 replica container của Nginx service đang chạy trên server 1 và 2, thử kiểm tra `docker inspect` Nginx container trên `server-1`:

```json
//root@server-1: docker inspect my-nginx.1.opdskafvlrclhc38t0tcw5uup
...
"NetworkSettings": {
	"Networks": {
	  "ingress": {
	      "IPAMConfig": {
	          "IPv4Address": "10.0.0.6"
	      },
	      "Links": null,
	      "Aliases": [
	          "b5af8738eb30"
	      ],
	      "NetworkID": "rasrehzw9jfeuj97ivx65nj5h",
	      "EndpointID": "3cf548a835fb3829d9fcb7f71259bad7554483a12d829a53880e5126974b1e36",
	      "Gateway": "",
	      "IPAddress": "10.0.0.6",
	      "IPPrefixLen": 24,
	      "IPv6Gateway": "",
	      "GlobalIPv6Address": "",
	      "GlobalIPv6PrefixLen": 0,
	      "MacAddress": "02:42:0a:00:00:06",
	      "DriverOpts": null
	  },
	  "nginx-net": {
	      "IPAMConfig": {
	          "IPv4Address": "10.0.1.3"
	      },
	      "Links": null,
	      "Aliases": [
	          "b5af8738eb30"
	      ],
	      "NetworkID": "jtlbkwkmjz9wzrhcym45ivrxh",
	      "EndpointID": "513595f853f714fb62c90093272c8ec66ac55fcaa904ccb9143bee15ec194a71",
	      "Gateway": "",
	      "IPAddress": "10.0.1.3",
	      "IPPrefixLen": 24,
	      "IPv6Gateway": "",
	      "GlobalIPv6Address": "",
	      "GlobalIPv6PrefixLen": 0,
	      "MacAddress": "02:42:0a:00:01:03",
	      "DriverOpts": null
	  }
	}
}
...
```

Bỏ qua `nginx-net` network, chúng ta sẽ tìm hiểu về **internal overlay network** ở phần sau. Thấy được container `my-nginx.1.opdskafvlrclhc38t0tcw5uup` chạy trên `server-1` đang được gắn với `ingress` network có địa chỉ IP là `10.0.0.6`.  Chúng ta tiếp tục kiểm tra `docker network inspect ingress`:

```json
//root@server-1: docker network inspect ingress
...
"IPAM": {
    "Driver": "default",
    "Options": null,
    "Config": [
        {
            "Subnet": "10.0.0.0/24",
            "Gateway": "10.0.0.1"
        }
    ]
},
...
"Containers": {
    "b5af8738eb30228c00423e6917d121e25c69222378c22839ea0fcc420a136f8f": {
        "Name": "my-nginx.1.opdskafvlrclhc38t0tcw5uup",
        "EndpointID": "3cf548a835fb3829d9fcb7f71259bad7554483a12d829a53880e5126974b1e36",
        "MacAddress": "02:42:0a:00:00:06",
        "IPv4Address": "10.0.0.6/24",
        "IPv6Address": ""
    },
    "ingress-sbox": {
        "Name": "ingress-endpoint",
        "EndpointID": "8e376885ef5c9be0bc56dae845dec3e5c55e1cc34df8964eec5a5db925acc71a",
        "MacAddress": "02:42:0a:00:00:02",
        "IPv4Address": "10.0.0.2/24",
        "IPv6Address": ""
    }
},
...
```

Có thể thấy `ingress network` thuộc dải IP `10.0.0.0/24` và hiện tại có 2 docker container đang được gắn với network này. `my-nginx.1.opdskafvlrclhc38t0tcw5uup` container hiển nhiên được gắn với ingress network vì chúng ta đã publish port  với `my-nginx` service, vậy `ingress-sbox` container kia từ đâu ra?

`ingress-sbox` là một hidden container được tạo ra mặc định (check trên cả 03 server đều thấy container này mặc dù `docker ps` không thấy chúng đâu cả), kiểm tra tiếp ta sẽ thấy `ingress-sbox` cũng được gắn với `docker_gwbridge network`:

```json
//root@server-1: docker network inspect docker_gwbridge
...
"IPAM": {
    "Driver": "default",
    "Options": null,
    "Config": [
        {
            "Subnet": "172.19.0.0/16",
            "Gateway": "172.19.0.1"
        }
    ]
},
...
"Containers": {
    "b5af8738eb30228c00423e6917d121e25c69222378c22839ea0fcc420a136f8f": {
        "Name": "gateway_c3db260d3ae0",
        "EndpointID": "11333a766b84c9a85e032d62c75895c23216c8a1115445118af593e8a50d24fa",
        "MacAddress": "02:42:ac:13:00:03",
        "IPv4Address": "172.19.0.3/16",
        "IPv6Address": ""
    },
    "ingress-sbox": {
        "Name": "gateway_ingress-sbox",
        "EndpointID": "6a9a17ae6df0ae39eac203b257fed6db70a4708e76f810d48fcbaa5cd766f662",
        "MacAddress": "02:42:ac:13:00:02",
        "IPv4Address": "172.19.0.2/16",
        "IPv6Address": ""
    }
},
...
```

Tổng hợp lại các thông tin về `ingress network`, `docker_gwbridge network` và `nginx container` trên cả 03 server ta sẽ có diagram như sau: 

![https://firebasestorage.googleapis.com/v0/b/knowledge-data.appspot.com/o/network-structure-ingress.jpg?alt=media](https://firebasestorage.googleapis.com/v0/b/knowledge-data.appspot.com/o/network-structure-ingress.jpg?alt=media)

Dựa vào diagram thấy được request từ client vào bất kỳ server nào trong 03 server trên đều được `iptables` forward vào IP (thuộc dải `docker_gwbridge` network) của `ingress-sbox` container: 

- `docker_gwbridge` là network được tạo tự động mặc định riêng trên mỗi server, nhiệm vụ của network này giống như với `docker0` (bridge network) vậy. Giúp cho máy host (server-3) có thể giao tiếp với các container, thử `curl 172.18.0.2:8080` ngay trên server-3 ta cũng nhận được kết quả tương tự như client nhận được.
- Vậy nên bản chất khi chúng ta truy cập vào địa chỉ của server-3 với port 8080, yêu cầu truy cập sẽ được NAT tới địa chỉ IP `172.18.0.2:8080`. Kiểm tra bảng `NAT` trong iptables bằng command `iptables -vL -t nat` sẽ thấy!
- Vì có `docker_gwbridge` network mà request từ người dùng có thể  đi vào hidden container là `ingress-sbox` rồi mới đi tới được service đích trong cụm Docker Swarm thông qua `ingress network` (như hình).

### Vậy làm sao từ `ingress-sbox` có thể lựa chọn được service đích (nginx-1 hoặc nginx-2) và thực hiện LB tới chúng như thế nào?

`**ingress-sbox` container được dùng để làm gì?**

- `ingress-sbox` đóng vai trò trung gian giữa máy host và ingress network,  các yêu cầu từ trong container này sẽ được tiếp tục chuyển tiếp tới địa chỉ **VIP (10.0.0.5)** trên `ingress network` (Docker Swarm sử dụng IPVS để cân bằng tải, đọc thêm về IPVS [ở đây](https://cloudcraft.info/loadbalance-gioi-thieu-lvs-ipvs/)).
- Địa chỉ **VIP** sẽ được tạo ra tương ứng với một dịch vụ triển khai có publish port ra ngoài, nghĩa là mỗi dịch vụ triển khai sẽ có địa chỉ **VIP** tương ứng để **IPVS** cân bằng tải các replica container trong dịch vụ đó (ví dụ trong bài này chính là dịch vụ `my-nginx` đã triển khai ở trên)
- Để kiểm tra địa chỉ **VIP** của một dịch vụ, sử dụng command `docker service inspect my-nginx` (chạy trên manager node):

    ```json
    ...
    "Endpoint": {
        "Spec": {
            "Mode": "vip",
            "Ports": [
                {
                    "Protocol": "tcp",
                    "TargetPort": 80,
                    "PublishedPort": 8080,
                    "PublishMode": "ingress"
                }
            ]
        },
        "Ports": [
            {
                "Protocol": "tcp",
                "TargetPort": 80,
                "PublishedPort": 8080,
                "PublishMode": "ingress"
            }
        ],
        "VirtualIPs": [
            {
                "NetworkID": "rasrehzw9jfeuj97ivx65nj5h",
                "Addr": "10.0.0.5/24"
            },
            {
                "NetworkID": "jtlbkwkmjz9wzrhcym45ivrxh",
                "Addr": "10.0.1.2/24"
            }
        ]
    }
    ...
    ```

**IPVS hoạt động ra sao trong Docker Swarm?**

Bởi vì `ingress-sbox` là hidden container nên chúng ta không thể exec vào trong để kiểm tra nó được, sử dụng `nsenter` để switch namespace ra ngoài máy host như sau:

```bash
➜  sudo nsenter --net=/run/docker/netns/ingress_sbox

### Có thể back trở lại namespace như ban đầu bằng command:
➜  sudo nsenter --net=/run/docker/netns/default
```

Tiếp tục chạy command dưới đây để xem thông tin `iptables` rules trong `ingress-sbox` namespace (Chú ý đến POSTROUTING Chain):

```bash
➜  iptables-save 
# Generated by iptables-save v1.6.1 on Wed Jul 21 21:09:34 2021
*mangle
:PREROUTING ACCEPT [1153:100216]
:INPUT ACCEPT [637:48400]
:FORWARD ACCEPT [516:51816]
:OUTPUT ACCEPT [637:48400]
:POSTROUTING ACCEPT [1153:100216]
-A PREROUTING -p tcp -m tcp --dport 8080 -j MARK --set-xmark 0x100/0xffffffff
**-A INPUT -d 10.0.0.5/32 -j MARK --set-xmark 0x100/0xffffffff**
COMMIT
# Completed on Wed Jul 21 21:09:34 2021
# Generated by iptables-save v1.6.1 on Wed Jul 21 21:09:34 2021
*filter
:INPUT ACCEPT [1432:115850]
:FORWARD ACCEPT [938:91965]
:OUTPUT ACCEPT [1432:115850]
COMMIT
# Completed on Wed Jul 21 21:09:34 2021
# Generated by iptables-save v1.6.1 on Wed Jul 21 21:09:34 2021
*nat
:PREROUTING ACCEPT [145:7276]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:DOCKER_OUTPUT - [0:0]
:DOCKER_POSTROUTING - [0:0]
-A OUTPUT -d 127.0.0.11/32 -j DOCKER_OUTPUT
-A POSTROUTING -d 127.0.0.11/32 -j DOCKER_POSTROUTING
**-A POSTROUTING -d 10.0.0.0/24 -m ipvs --ipvs -j SNAT --to-source 10.0.0.4**
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p tcp -m tcp --dport 53 -j DNAT --to-destination 127.0.0.11:33731
-A DOCKER_OUTPUT -d 127.0.0.11/32 -p udp -m udp --dport 53 -j DNAT --to-destination 127.0.0.11:39025
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p tcp -m tcp --sport 33731 -j SNAT --to-source :53
-A DOCKER_POSTROUTING -s 127.0.0.11/32 -p udp -m udp --sport 39025 -j SNAT --to-source :53
COMMIT
# Completed on Wed Jul 21 21:09:34 2021
```

Trong `mangle` table của `iptables` sẽ đánh dấu (MARK) các gói tin có Destination IP là `10.0.0.5` với `0x100`, mà `10.0.0.5` chính là địa chỉ **VIP** đã đề cập ở mục trên. Vậy các gói tin được đánh dấu với `0x100` sẽ đi đâu về đâu? Thực hiện kiểm tra tiếp sử dụng `ipvsadm` (tool được dùng để cấu hình IPVS) với command `ipvsadm -L -n --stats` để xem thông tin cấu hình hiện tại:

```bash
➜  ipvsadm -L -n --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
FWM  256                               266     1175      945    90915    92590
  -> 10.0.0.6:0                        133      552      447    43073    43507
  -> 10.0.0.7:0                        133      623      498    47842    49083
```

Đến đây thì quá rõ luôn rồi 😀  những gói tin được đánh dấu với MARK (`0x100` convert to decimal is `256`) sẽ được cân bằng tải tới 02 địa chỉ IP là `10.0.0.6` và `10.0.0.7` Xem lại diagram phía trên thì thấy rằng 02 địa chỉ này chính là địa chỉ của Nginx container chạy trên server-1 và server-2.

![https://firebasestorage.googleapis.com/v0/b/knowledge-data.appspot.com/o/ingress-lb.png?alt=media](https://firebasestorage.googleapis.com/v0/b/knowledge-data.appspot.com/o/ingress-lb.png?alt=media)

> Để ý thêm trong rule NAT của `iptables` (POSTROUTING Chain) sẽ có thêm đoạn các gói packets sẽ được NAT lại Source IP là `10.0.0.4`, mà `10.0.0.4` lại chính là địa chỉ IP của `ingress-sbox` trên ingress network. Việc này đảm bảo sau khi packets khi gửi đến dịch vụ đích thì sẽ phản hồi  lại cho `ingress-sbox` (đại diện cho server-2)rồi từ đó phản hồi lại cho client ([đọc thêm](https://www.haproxy.com/blog/layer-4-load-balancing-nat-mode/) về Load Balancing Layer 4 sử dụng cơ chế NAT)

## Internal Overlay Network hoạt động như thế nào?

Đối với Internal Overlay Network thì cơ chế routing mesh (load balancer) không khác gì so với phần Ingress Network đã giải thích phía trên. Để hình dung rõ hơn về giao tiếp nội bộ giữa các service chúng ta triển khai thêm backend service như sau:

```bash
➜  docker service create \
  --name backend \
  --replicas=1 \
  --network nginx-net \
  thuongnn1997/nginx:debug
```

Triển khai thêm backend service cũng được join với `nginx-net` network đã tạo ở ví dụ trước. Thực hiện `docker inspect` lấy các thông tin như ở phần trước, ta cũng có diagram của các service giao tiếp với `nginx-net` network như sau:

![https://firebasestorage.googleapis.com/v0/b/knowledge-data.appspot.com/o/internal-overlay-network.jpg?alt=media](https://firebasestorage.googleapis.com/v0/b/knowledge-data.appspot.com/o/internal-overlay-network.jpg?alt=media)

- Khác biệt với Ingress Network, Internal Overlay Network (nginx-net) sẽ không có chân nào cho phép từ bên ngoài có thể kết nối vào được bên trong network này.
- Docker Swarm cho phép chúng ta tạo overlay network bất kỳ và chúng hoàn toàn private, chỉ có những service bên trong network này mới có thể giao tiếp được với nhau.
- Không có cách nào có thể truy cập vào được mạng này từ bên ngoài như máy host, việc publish một dịch vụ ra ngoài thì cần phải sử dụng Ingress Network (trong command chỉ cần thêm publish port khi khởi tạo service)

---

Thực hiện `docker exec` vào container của backend service trên server-3, vì `backend` service và `my-nginx` service đều dùng chung `nginx-net` network nên đứng từ backend container ta có thể kết nối được tới các container của `my-nginx` service như sau:

```bash
### docker exec -it backend.1.zchpr6x6hvr6tofqlpl2vob3z /bin/bash
➜  curl my-nginx
Welcome to nginx-2 from server-2
➜  curl my-nginx
Welcome to nginx-1 from server-1
➜  curl my-nginx
Welcome to nginx-2 from server-2
```

Request từ backend container sẽ được LB tới các container của `my-nginx service` (nội dung trả về từ cả nginx-1 và nginx-2 sau mỗi lần request). Bản chất mỗi request từ backend container đều được đẩy tới địa chỉ **VIP** của `my-nginx service` (10.0.1.2), kiểm tra lại địa chỉ IP được resolve từ domain `my-nginx` sẽ thấy:

```bash
➜  nslookup my-nginx
Server:         127.0.0.11
Address:        127.0.0.11#53

Non-authoritative answer:
Name:   my-nginx
Address: 10.0.1.2
```

Địa chỉ IP được resolve từ `my-nginx` không phải là của container `nginx-1` (10.0.1.3) hay `nginx-2` (10.0.1.4) mà chính là địa chỉ **VIP**, tương tự như cách hoạt động của **IPVS** như đã tìm hiểu ở phần Ingress Network. Chúng ta lại tiếp sử dụng `nsenter` tool để switch sang namespace của `lb-nginx-net` trên server-3 thấy được như sau:

![https://firebasestorage.googleapis.com/v0/b/knowledge-data.appspot.com/o/internal-overlay-network-flow.png?alt=media](https://firebasestorage.googleapis.com/v0/b/knowledge-data.appspot.com/o/internal-overlay-network-flow.png?alt=media)

## Bypass the routing mesh (IVPS)

- Qua tìm hiểu về Ingress và Internal Overlay Network ở trên, chúng ta thấy được cả 02 loại network này áp dụng cơ chế routing mesh sử dụng **IPVS** để cân bằng tải các yêu cầu tới các replica container.
- Đặc điểm của **IPVS** (NAT mode) là sẽ thay đổi thông tin (Source, Destination IP) của gói tin qua lại con Load Balancer (trên 02 ví dụ trên lần lượt là `ingress-sbox` và `lb-nginx-net`).
- Ngoài Virtual IP (VIP) mode, Docker Swarm còn hỗ trợ DNS Round Robin (DNSRR) mode bằng cách cấu hình `-endpoint-mode=dnsrr`.

### DNS Round Robin hoạt động như thế nào?

Triển khai một dịch vụ khác sử dụng DNSRR mode bằng command dưới đây:

```bash
➜  docker network create -d overlay demo-net
➜  docker service create \
  --name user-api \
  --publish target=80,published=8081,mode=host \
  --replicas=3 \
  --endpoint-mode=dnsrr \
  --network demo-net \
  thuongnn1997/nginx:debug
```

Triển khai `user-api` service trên overlay network là `demo-net`, tham số `--endpoint-mode` để là `dnsrr` và `mode=host` trong tham số publish (bắt buộc khi triển khai với DNSRR mode).

Sau khi triển khai xong, hãy thử `docker exec` vào một container bất kỳ đã được triển khai trong cụm Swarm. Truy cập vào dịch vụ thông qua domain `user-api` ta vẫn nhận được kết quả giống với **VIP** mode (không có gì khác biệt ở đây):

```bash
### docker exec -it user-api.1.o6cu6rp3jqje8ay0drvgqnchi /bin/bash
➜  curl user-api   
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Tuy nhiên hãy thử kiểm tra địa chỉ IP được resolve từ domain `user-api` như sau:

```bash
➜  nslookup user-api
Server:         127.0.0.11
Address:        127.0.0.11#53

Non-authoritative answer:
Name:   user-api
Address: 10.0.3.4
Name:   user-api
Address: 10.0.3.3
Name:   user-api
Address: 10.0.3.2
```

Điểm khác biệt chính là ở đây! nếu như với `VIP mode` sẽ chỉ resolve ra một địa chỉ duy nhất là địa chỉ **VIP**  thì với `DNSRR mode` sẽ trả về luôn danh sách các địa chỉ IP của replica container.

- Load Balancing đối với `VIP mode` sẽ đóng vai trò forwarder chuyển tiếp các packets đến container đích hợp lý. Mọi packets qua lại đều phải đi qua LB.
- Load Balancing đối với `DNSRR mode` thay vì chuyển tiếp packets thì nó sẽ trả về luôn danh sách địa chỉ IP của container đích. Đặc biệt là danh sách này được đảo đều (Round Robin) sau mỗi lần request.
- DNSRR mode chỉ trả về danh sách IP, việc kết nối và trao đổi packets sẽ diễn ra trực tiếp với container đích mà không cần phải đi qua LB nữa.

### Vậy khi nào thì sử dụng DNS Round Robin?

![https://docs.docker.com/engine/swarm/images/ingress-lb.png](https://docs.docker.com/engine/swarm/images/ingress-lb.png)

Trong thực tế khi chạy hệ thống trên production, chúng ta vẫn cần một External IP cấu hình domain để client có thể kết nối vào hệ thống.

- Bởi vì nếu setup kết nối thẳng vào một trong 03 node trên sẽ có khả năng một node bị down (đúng ngay node cấu hình domain name) khiến cho client không thể gửi request được nữa.
- Vậy nên đằng trước Swarm Cluster thường sẽ thiết kế một External Load Balancer để nhận request từ client rồi cân bằng tải tới một trong các node trong cụm (như hình trên).
- Tuy nhiên trên Swarm node mặc định sử dụng `routing mesh` (sử dụng IPVS để cân bằng tải) như đã giải thích ở phần **Ingress network hoạt động như thế nào?**
- Mặc định routing mesh đẩy các request tới `swarm load balancer` để cân bằng tải (sử dụng NAT packets) tới service đích phù hợp (có thể service đích không tồn tại trên node đó), điều này có thể làm tăng thời gian request từ client tới dịch vụ.

---

Từ vấn đề trên trong một số hệ thống có thể cân nhắc bỏ swarm load balancer đi để đạt được thời gian truy vấn tối ưu hơn, lúc này cấu hình **DNSRR** cho LB sẽ được sử dụng. Tuy nhiên sử dụng mode này chúng ta sẽ có những đánh đổi dưới đây:

- Các replica container sẽ chạy dưới publish mode là host, tương tự như chúng ta chạy một container thông thường với network là host.
- Việc này đồng nghĩa rằng chúng ta không thể chạy nhiều service task trên mỗi Swarm node (ví dụ như chúng ta có 3 server nodes nhưng chạy tận 6 replicas), ta cũng không thể chỉ định target port cho từng service task được.
- Có 02 cách để chạy nhiều service task hơn số lượng nodes là:
    - Để Docker tự động listen random port cho service task trên server node (bỏ tham số `published` khi triển khai), lúc này mỗi service task sẽ có listen một port khác nhau nên các server nodes không còn bị trùng port nữa.
    - Tất cả service task vẫn listen chung một port, tuy nhiên đảm bảo số lượng replica tương ứng với số server nodes. Sử dụng tham số `—mode` với giá trị là `global` khi triển khai sẽ tự động triển khai 1 service task tương ứng mỗi server node.
- Nếu triển khai số lượng service task nhỏ hơn số lượng server nodes, sẽ xảy ra một vấn đề là nếu client truy cập vào 1 server node mà trên đó không có service task nào đang chạy. Vì không có routing mesh nào được sử dụng ở đây nên client sẽ không nhận được phản hồi như `VIP mode`.

# Tài liệu tham khảo

[https://docs.docker.com/network/overlay](https://docs.docker.com/network/overlay/)

[https://docs.docker.com/engine/swarm/ingress](https://docs.docker.com/engine/swarm/ingress/)

[https://github.com/sbrattla/docs/blob/master/DockerInternalLoadBalancing.md](https://github.com/sbrattla/docs/blob/master/DockerInternalLoadBalancing.md)