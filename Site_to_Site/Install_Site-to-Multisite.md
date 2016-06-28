# Cấu hình OpenVPN Site to MultiSite sử dụng Routing-tunel
## Mục lục
[1.Mô hình mạng] (#1)

[2.Giới thiệu] (#2)

[3.Các bước triển khai] (#3)

[4.Cấu hình chi tiết] (#4)

[5.Tham Khảo] (#5)

<a name="1"></a>
### 1.Mô hình mạng
<img src="http://image.prntscr.com/image/d46b0ea22c244988ae645ea72770b389.png" />

<a name="2"></a>
### 2.Giới thiệu
```sh
--------------------------------------------------------------
OpenVPN Server1
--------------------------------------------------------------
IP Wan	:		192.168.100.10
IP Lan  :   192.168.10.254
IP tunel1 + tunel2:   10.10.10.1
port tunel1: 1194
port tunel2: 1195
OS				:		Centos 6.7 Final
```

```sh
--------------------------------------------------------------
OpenVPN Server2
--------------------------------------------------------------
IP Wan	:		192.168.100.11
IP Lan  :   192.168.20.254
IP tunel1:   10.10.10.2
port tunel1: 1194
OS				:		Centos 6.7 Final
```

```sh
--------------------------------------------------------------
OpenVPN Server3
--------------------------------------------------------------
IP Wan	:		192.168.100.12
IP Lan  :   192.168.30.254
IP tunel2:   10.10.10.3
port tunel2: 1195
OS				:		Centos 6.7 Final
```

```sh
--------------------------------------------------------------
VPN Client1 bên trong mạng Lan của Server1
--------------------------------------------------------------
IP address		:		192.168.10.10
OS				:		Windows 7
```

```sh
--------------------------------------------------------------
VPN Client2 bên trong mạng Lan của Server2
--------------------------------------------------------------
IP address		:		192.168.20.10
OS				:		Windows 7
```

```sh
--------------------------------------------------------------
VPN Client3 bên trong mạng Lan của Server3
--------------------------------------------------------------
IP address		:		192.168.30.10
OS				:		Windows 7
```

<a name="3"></a>
### 3.Các bước triển khai
- Cấu hình VPN trên VPNServer1 (role Server)
- Cấu hình VPN trên VPNServer2 (role Client1)
- Cấu hình VPN trên VPNServer3 (role Client2)

<a name="4"></a>
### 4.Cấu hình chi tiết
#### 4.1.Cấu hình VPN trên Server1 (role Server)
- Tiến hành cài đặt và tạo các key giống như trong bài https://github.com/kieulam141/OpenVPN/blob/master/CA_Certificate_Keys.md.
- Thay đổi 1 chút là tạo key server tên "Server1" , key client1 tên "Server2", key client2 tên "Server3"
- Do ta phải tạo 2 tunel kết nối đến 2 miền server khác nhau, nên trong thư mục /etc/openvpn chúng ta nên tạo cho mỗi tunel 1 file config riêng cho nó
để dễ quản lý cũng như cấu hình.
- Cấu hình file conf cho tunel1: `vi /etc/openvpn/vpn1.conf`
```sh
dev tun
remote 192.168.100.11     #IP của Server2
ifconfig 10.10.10.1 10.10.10.2  # 10.10.10.1:IP tunel1 của Server1, 10.10.10.2:IP tunel1 của Server2
up /etc/openvpn/scripts/routes.up.sh  #chạy script để routing mạng lan bên kia(Server2)

tls-server
daemon

# Diffie-Hellman Parameters (tls-server only)
dh /etc/openvpn/keys/dh2048.pem

# Certificate Authority file
ca /etc/openvpn/keys/ca.crt

# Our certificate/public key
cert /etc/openvpn/keys/Server1.crt

# Our private key
key /etc/openvpn/keys/Server2.key

reneg-sec 300

port 1194

# Verbosity level.
# 0 -- quiet except for fatal errors.
# 1 -- mostly quiet, but display non-fatal network errors.
# 3 -- medium output, good for normal operation.
# 9 -- verbose, good for troubleshooting
verb 3
```

- Tương tự là file conf cho tunel2: `vi /etc/openvpn/vpn2.conf`
```sh
dev tun
remote 192.168.100.12     #IP của Server3
ifconfig 10.10.10.1 10.10.10.3  # 10.10.10.1:IP tunel2 của Server1, 10.10.10.3:IP tunel2 của Server3
up /etc/openvpn/scripts/routes2.up.sh  #chạy script để routing mạng lan bên kia(Server3)

tls-server
daemon

# Diffie-Hellman Parameters (tls-server only)
dh /etc/openvpn/keys/dh2048.pem

# Certificate Authority file
ca /etc/openvpn/keys/ca.crt

# Our certificate/public key
cert /etc/openvpn/keys/Server1.crt

# Our private key
key /etc/openvpn/keys/Server2.key

reneg-sec 300

port 1194

# Verbosity level.
# 0 -- quiet except for fatal errors.
# 1 -- mostly quiet, but display non-fatal network errors.
# 3 -- medium output, good for normal operation.
# 9 -- verbose, good for troubleshooting
verb 3
```

- Tương tự khi ta muốn routing từ Server1 tới các dải Lan bên trong Server2 và Server3 thì ta cũng tạo riêng các scripts để routing
từng dải Lan bên trong 2 miền Server đấy.
- Cấu hình script routing Lan bên trong Server2: vi /etc/openvpn/scripts/routes.up.sh
```sh
#!/bin/sh

/sbin/route add -net 192.168.20.0 netmask 255.255.255.0 gw 10.10.10.1
```
- Cấu hình script routing Lan bên trong Server3: vi /etc/openvpn/scripts/routes2.up.sh
```sh
#!/bin/sh

/sbin/route add -net 192.168.30.0 netmask 255.255.255.0 gw 10.10.10.1
```
- Phân quyền cho scrpit: `chmod 700 /etc/openvpn/scripts/routes.up.sh /etc/openvpn/scripts/routes2.up.sh `
- Cấu hình iptables : 
```sh
iptables -A INPUT -i eth0 -m udp -p udp -s 192.168.100.10 --dport 1194 -j ACCEPT        #eth0: card wan, forward port 1194
iptables -A INPUT -i eth0 -m udp -p udp -s 192.168.100.10 --dport 1195 -j ACCEPT        # forward port 1195
iptables -A FORWARD -i eth1 -o tun+ -j ACCEPT                                         #eth1: card lan
iptables -A FORWARD -i tun+ -o eth1 -j ACCEPT                                         #tun+ = tun0 + tun1 + ...
```
- Khởi động dịch vụ : `service openvpn start`

#### 4.2.Cấu hình VPN trên Server2 (role Client1)
- Chỉ tiến hành cài đặt dịch vụ OpenVPn mà không cần tạo các key hay certificate.
- Copy các key và certificate mà Server1 đã tạo ra ở trên về đường dẫn /etc/openvpn/keys/
- Cấu hình file conf: vi /etc/sysconfig/server.conf
```sh
dev tun
remote 192.168.100.10
ifconfig 10.10.10.2 10.10.10.1
up /etc/openvpn/scripts/routes.up.sh

tls-client
remote-cert-tls server
daemon

# Certificate Authority file
ca /etc/openvpn/keys/ca.crt

# Our certificate/public key
cert /etc/openvpn/keys/Server2.crt

# Our private key
key /etc/openvpn/keys/Server2.key

reneg-sec 300

port 1194

# Verbosity level.
# 0 -- quiet except for fatal errors.
# 1 -- mostly quiet, but display non-fatal network errors.
# 3 -- medium output, good for normal operation.
# 9 -- verbose, good for troubleshooting
verb 3
```

- Cấu hình script routing: vi /etc/openvpn/scripts/routes.up.sh
```sh
#!/bin/sh

/sbin/route add -net 192.168.10.0 netmask 255.255.255.0 gw 10.10.10.2
```
- phân quyền cho scrpit: `chmod 700 /etc/openvpn/scripts/routes.up.sh`
- Cấu hình iptables : 
```sh
iptables -A INPUT -i eth0 -m udp -p udp -s 192.168.100.11 --dport 1194 -j ACCEPT        #eth0: card wan, port 1194
iptables -A FORWARD -i eth1 -o tun0 -j ACCEPT                                         #eth1: card lan
iptables -A FORWARD -i tun0 -o eth1 -j ACCEPT
```
- Khởi động dịch vụ : `service openvpn start`

#### 4.3.Cấu hình VPN trên Server2 (role Client)
- Chỉ tiến hành cài đặt dịch vụ OpenVPn mà không cần tạo các key hay certificate.
- Copy các key và certificate mà Server1 đã tạo ra ở trên về đường dẫn /etc/openvpn/keys/
- Cấu hình file conf: vi /etc/sysconfig/server.conf
```sh
dev tun
remote 192.168.100.10
ifconfig 10.10.10.3 10.10.10.1
up /etc/openvpn/scripts/routes.up.sh

tls-client
remote-cert-tls server
daemon

# Certificate Authority file
ca /etc/openvpn/keys/ca.crt

# Our certificate/public key
cert /etc/openvpn/keys/Server3.crt

# Our private key
key /etc/openvpn/keys/Server3.key

reneg-sec 300

port 1194

# Verbosity level.
# 0 -- quiet except for fatal errors.
# 1 -- mostly quiet, but display non-fatal network errors.
# 3 -- medium output, good for normal operation.
# 9 -- verbose, good for troubleshooting
verb 3
```

- Cấu hình script routing: vi /etc/openvpn/scripts/routes.up.sh
```sh
#!/bin/sh

/sbin/route add -net 192.168.10.0 netmask 255.255.255.0 gw 10.10.10.3
```
- phân quyền cho scrpit: `chmod 700 /etc/openvpn/scripts/routes.up.sh`
- Cấu hình iptables : 
```sh
iptables -A INPUT -i eth0 -m udp -p udp -s 192.168.100.12 --dport 1195 -j ACCEPT        #eth0: card wan, port 1195
iptables -A FORWARD -i eth1 -o tun1 -j ACCEPT                                         #eth1: card lan
iptables -A FORWARD -i tun1 -o eth1 -j ACCEPT                                         #tunel1: giống với của Server1
```
- Khởi động dịch vụ : `service openvpn start`
- Cuối cùng tiến hành ping từ lan trong Server1 sang lan trong Server2 và Server3, lưu ý xem firewall trên các client đã disable hoặc đã tạo rule để 
forward các gói tin hay chưa.Hoặc là selinux trên các Server đã tắt hay chưa.
