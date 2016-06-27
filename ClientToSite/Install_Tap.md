# Cấu Hình OpenVPN Client-to-Site trên Centos7 sử dụng Tap-Bridging
## Mục lục
[1.Mô hình mạng] (#1)

[2.Giới thiệu] (#2)

[3.Các bước triển khai] (#3)

[4.Cấu hình chi tiết] (#4)

[5.Tham Khảo] (#5)

<a name="1"></a>
### 1.Mô hình mạng
<img src="http://image.prntscr.com/image/bb522804894b4c7aa47dea9e1580aebb.png" />

<a name="2"></a>
### 2.Giới thiệu
```sh
--------------------------------------------------------------
OpenVPN Server
--------------------------------------------------------------
IP Wan		:		192.168.100.10
IP Lan  :  192.168.20.254
OS				:		Centos 7 Final
```

```sh
--------------------------------------------------------------
VPN Client (bên ngoài)
--------------------------------------------------------------
IP Wan		:		192.168.100.17
OS				:		Windows 7
```

```sh
--------------------------------------------------------------
VPN Client (bên trong mạng lan)
--------------------------------------------------------------
IP address		:		192.168.20.10
OS				:		Windows 7
```

<a name="3"></a>
### 3.Các bước triển khai
- Enable the epel-repository in CentOS.
- Install openvpn, easy-rsa .
- Configure easy-rsa.
- Configure openvpn.
- Start openVPN Server.
- Setting up the OpenVPN client application
- Install OpenVPN Client on Linux

<a name="4"></a>
### 4.Cấu hình chi tiết
#### 4.1.Enable the epel-repository
- `rpm -ivh http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-6.noarch.rpm`

#### 4.2.Install openvpn, easy-rsa
- `yum -y install openvpn easy-rsa`

- Sau khi quá trình cài đặt hoàn tất , chép file server.conf đến thư mục /etc/openvpn/
```sh
# cd /usr/share/doc/openvpn-*/sample/sample-config-files/
# cp server.conf /etc/openvpn
```
#### 4.3.Configure easy-rsa
- Tạo Keys và Certificates
```sh
# mkdir /etc/openvpn/rsa
# cp –rf /usr/share/easy-rsa/2.0/* /etc/openvpn/rsa
```
- Cấu hình file vars.Thay đổi tùy theo ý của bạn
```sh
vi /etc/openvpn/rsa/vars
```
```sh
export KEY_COUNTRY="VN"
export KEY_PROVINCE="HN"
export KEY_CITY="HN"
export KEY_ORG="Meditech"
export KEY_EMAIL="kieutunglam@gmail.com"
export KEY_OU="Meditech"
```
```sh
# cd /etc/openvpn/rsa/
# source ./vars
```
- Chạy build-ca scripts để tạo Certificates + keys.Nếu Không muốn chỉnh sửa gì thì các bạn nhập "Enter" hết khi được hỏi.
```sh
# ./build-ca
```
- Tiếp theo tạo Key và Certificate cho server.Nếu Không muốn chỉnh sửa gì thì các bạn nhập "Enter" hết khi được hỏi.
```sh
# ./build-key-server server
```
- Tạo file trao đổi key Diffie-Hellman
```sh
# ./build-dh
```
- Tạo Key và Certificate cho Client.Nếu Không muốn chỉnh sửa gì thì các bạn nhập "Enter" hết khi được hỏi.
```sh
# ./build-key win7
```

#### 4.4.Configure OpenVPN Server
```sh
# vi /etc/openvpn/server.conf
```
```sh
# line 35: uncomment tcp and comment out udp
proto tcp
;proto udp

# line 52: thay bằng tap để chạy bridge mode
dev tap0
;dev tun

# line 78: đổi đường dẫn cho certificates.
ca /etc/openvpn/rsa/keys/ca.crt
cert /etc/openvpn/rsa/keys/server.crt
key /etc/openvpn/rsa/keys/server.key  # This file should be kept secret

dh /etc/openvpn/rsa/keys/dh2048.pem

# line 101: comment out
;server 10.8.0.0 255.255.255.0

# line 120: uncomment and thay ⇒ [VPN server's IP] [subnetmask] [the range of IP for client]
server-bridge 192.168.20.0 255.255.255.0 192.168.20.100 192.168.20.200

# line 289: uncomment and chỉ định đường dẫn file log.
log /var/log/openvpn.log
log-append /var/log/openvpn.log
```

- Copy các file để khởi động cũng như dừng openvpn-bridge
```sh
cp /usr/share/doc/openvpn-*/sample/sample-scripts/bridge-start /etc/openvpn/openvpn-startup 
cp /usr/share/doc/openvpn-*/sample/sample-scripts/bridge-stop /etc/openvpn/openvpn-shutdown 
chmod 755 /etc/openvpn/openvpn-startup /etc/openvpn/openvpn-shutdown 
vi /etc/openvpn/openvpn-startup
```

```sh
# line 17-20: change
eth="eth1" # card mạng để nối bridge
eth_ip="192.168.20.254"# IP for bridge interface
eth_netmask="255.255.255.0"# subnet mask
eth_broadcast="192.168.20.255"# broadcast address

# add follows to the end: define gateway
eth_gw="192.168.20.254" #gateway chính là địa chỉ của router, nhưng trong trường hợp, VPNServer chính là gateway.
route add default gw $eth_gw
```
- cấu hình bridge
```sh
cp /usr/lib/systemd/system/openvpn@.service /usr/lib/systemd/system/openvpn-bridge.service 
vi /usr/lib/systemd/system/openvpn-bridge.service
```

```sh
# change like follows in [Service] section
[Service]
PrivateTmp=true
Type=forking
PIDFile=/var/run/openvpn/openvpn.pid  
ExecStartPre=/bin/echo 1 > /proc/sys/net/ipv4/ip_forward  # enable routing trên server
ExecStartPre=/etc/openvpn/openvpn-startup   # chỉ định file conf khởi động bridge
ExecStart=/usr/sbin/openvpn --daemon --writepid /var/run/openvpn/openvpn.pid --cd /etc/openvpn/ --config server.conf
ExecStopPost=/etc/openvpn/openvpn-shutdown  # chỉ định file conf đề dừng dịch vụ bridge
ExecStopPost=/bin/echo 0 > /proc/sys/net/ipv4/ip_forward # disable routing trên server
```
- Khởi chạy bridge và enbale.
```sh
 systemctl start openvpn-bridge 
 systemctl enable openvpn-bridge 
 ```
- Cấu hình iptables : vi /etc/sysconfig/iptables
```sh
-A INPUT -p tcp -m tcp --dport 1194 -j ACCEPT
-A INPUT -i tap+ -j ACCEPT
-A FORWARD -i tap+ -j ACCEPT
-A FORWARD -i tap+ -o eth0 -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT  #eth0 card wan
-A FORWARD -i eth0 -o tap+ -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
```
- Cuối cùng copy các file "ca.crt, win7.crt, win7.key" ở thư mục /etc/openvpn/rsa/keys sang Client.

#### 4.5.Setting up the OpenVPN client application
- Sau khi copy các file trên sang client, thì tiến hành cài đặt OpenVPN client theo link: http://openvpn.net/index.php/open-source/downloads.html
- Sau khi cài đặt copy file client.openvpn ở đường dẫn "C:\Program Files\OpenVPN\sample-config" vào trong đường dẫn "C:\Program Files\OpenVPN\config" và bạn có thể đổi tên theo ý của bạn.
- Copy file "ca.crt", "win7.crt", "win7.key" vào chung đường dẫn với file config hoặc 1 folder nào đó, và phải nhớ đường dẫn để chỉ ra trong file config.
- Mở file client.ovpn và chỉnh sửa:
```sh
# it's OK with default
client
# device name which you specified in the server's config
dev tap0
;dev tun
# protocol which you specified in the server's config
proto tcp
;proto udp
# OpenVPN server's global IP abnd port (replace to your own environment)
remote 192.168.100.10 1194   #192.168.100.10: IP VPNServer
# retry resolving
resolv-retry infinite
# no bind for local port
nobind
# enable persist options
persist-key
persist-tun
# path for certificates
ca ca.crt   # Nếu bạn để các file certificate của client ở thư mục # thì phải chỉ rõ đường dẫn.
cert win7.crt
key win7.key
# enable compress
comp-lzo
# log level
verb 3
```
- Cuối cùng mở ứng dụng OpenVPN trên client với quyền admin và connect tới VPNServer, khi đó client sẽ được cấp ip trong dải 192.168.20.0/24.
- Ping đến địa chỉ bridge của VPNServer và địa chỉ của VPN client bên trong xem đã thành công hay chưa.

<a name="5"></a>
### 5.Tham Khảo
- http://www.server-world.info/en/note?os=CentOS_7&p=openvpn&f=1
