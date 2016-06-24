# Cấu Hình OpenVPN Client-to-Site trên Centos7 sử dụng routing-tunel
## Mục lục
[1.Mô hình mạng] (#1)

[2.Giới thiệu] (#2)

[3.Các bước triển khai] (#3)

[4.Cấu hình chi tiết] (#4)

[5.Tham Khảo] (#5)

<a name="1"></a>
### 1.Mô hình mạng
<img src="http://i.imgur.com/ODSh1q8.png" />

<a name="2"></a>
### 2.Giới thiệu
```sh
--------------------------------------------------------------
OpenVPN Server
--------------------------------------------------------------
IP address		:		192.168.100.13
OS				:		Centos 7 Final
```

```sh
--------------------------------------------------------------
VPN Client (bên ngoài)
--------------------------------------------------------------
IP address		:		192.168.100.9
OS				:		Windows 7
```

```sh
--------------------------------------------------------------
VPN Client (bên ngoài)
--------------------------------------------------------------
IP address		:		192.168.100.9
OS				:		Centos 6.7
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
vi /etc/openvpn//rsa/vars
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
- Tạo file dh2048.pem trong thư mục /etc/openvpn/rsa/keys/
```sh
# ./build-dh
```
- Tạo Key và Certificate cho Client.Nếu Không muốn chỉnh sửa gì thì các bạn nhập "Enter" hết khi được hỏi.
```sh
# ./build-key client
```

#### 4.4.Configure OpenVPN Server
```sh
# vi /etc/openvpn/server.conf
```
- chú ý đến đường dẫn các file ca.crt ,server.crt ,server.key
```sh
ca /etc/openvpn/rsa/keys/ca.crt
cert /etc/openvpn/rsa/keys/server.crt
key /etc/openvpn/rsa/keys/server.key  # This file should be kept secret

# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh2048.pem 2048
dh /etc/openvpn/rsa/keys/dh2048.pem
```
- Đường mạng sẽ cấp địa chỉ IP khi VPN Client kết nối đến VPN Server
```sh
# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 192.168.30.0 255.255.255.0
```
- Route đường mạng trên để có thể kết nối được tới Client bên trong mạng lan
```sh
# Push routes to the client to allow it
# to reach other private subnets behind
# the server.  Remember that these
# private subnets will also need
# to know to route the OpenVPN client
# address pool (10.8.0.0/255.255.255.0)
# back to the OpenVPN server.
;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"
push "route 192.168.20.0 255.255.255.0"
```
- Cho phép tất cả lưu lượng (như duyệt web và phân giải DNS )đi qua VPN bằng cách bỏ “;” tại dòng này.Nhưng khuyến cáo không nên để tham số này vì
client VPN bên ngoài khi truy cập Internet sẽ chiếm băng thông của VPN Server ra Internet.
```sh
push "redirect-gateway def1 bypass-dhcp"
```
- DNS để phân giải
```sh
# Certain Windows-specific network settings
# can be pushed to clients, such as DNS
# or WINS server addresses.  CAVEAT:
# http://openvpn.net/faq.html#dhcpcaveats
# The addresses below refer to the public
# DNS servers provided by opendns.com.
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
```
- Mặc định khi kết nối đến VPN thì client chỉ có thể nhìn thấy VPN Server mà không nhìn thấy các client # cùng kết nối đến, nếu bạn muốn
thì có thể bỏ ";" tại dòng này
```sh
# Uncomment this directive to allow different
# clients to be able to "see" each other.
# By default, clients will only see the server.
# To force clients to only see the server, you
# will also need to appropriately firewall the
# server's TUN/TAP interface.
;client-to-client
```

#### 4.5.Configure Iptables
- Install the iptables và disable firewalld dùng các câu lệnh sau
```sh
yum install iptables-services -y
systemctl mask firewalld
systemctl enable iptables
systemctl stop firewalld
systemctl start iptables
iptables --flush
```
- Tiếp theo add rules cho iptables để forward các định tuyến của kết nối (mạng 192.168.30.0) tới OpenVPN subnet (mạng 192.168.100.0)
```sh
iptables -I INPUT -i eth0 -m state --state NEW -p udp --dport 1194 -j ACCEPT 
# Allow TUN interface connections to OpenVPN server
iptables -I INPUT -i tun+ -j ACCEPT 
# Allow TUN interface connections to be forwarded through other interfaces
iptables -I FORWARD -i tun+ -j ACCEPT
iptables -I FORWARD -i tun+ -o eth0 -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
iptables -I FORWARD -i eth0 -o tun+ -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
iptables -t nat -A POSTROUTING -s 192.168.30.0/24 -o eth0 -j MASQUERADE // Dải LAN bên trên
```
- Bật Port Forwarding : thêm dòng `net.ipv4.ip_forward=1` vào file /etc/sysctl.conf

#### 4.6 Cấu hình cho client
- Trước khi cấu hình thì copy các file ca.crt, client.crt, client.key vào client.Nhớ để ý đường dẫn để khi cấu hình file conf cho client.
- File conf client
```sh
client
;dev tap0
dev tun
;proto tcp
proto udp
remote 192.168.100.17 1194
resolv-retry infinite
nobind
user nobody
group nobody
persist-key
persist-tun
ca ca.crt
cert client01.crt
key client01.key
remote-cert-tls server
comp-lzo
verb 3
route no-pull # Ra Internet bang gateway cua chinh client, chu khong qua VPN.
route 8.8.8.8
```
