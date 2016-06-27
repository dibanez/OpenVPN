# Thiết lập Certificate Authority (CA) , public keys và private keys cho OpenVPN server và các client
## Mục lục
[1.Giới thiệu tổng quan]	(#1)

[2.Thiết lập CA chủ,certificate và keys]	(#2)

[3.Tham Khảo] (#3)

<a name="1"></a>
### 1.Giới thiệu tổng quan
- Bước đầu tiên khi cấu hình OpenVPN là thiết lập 1 PKI (public key infrastructure).1 PKI bao gồm:
<ul>
	<li>Các chứng chỉ riêng biệt-certificate (hay còn gọi là key public) và private key cho server và mỗi client.</li>
	<li>1 CA chủ và key để đăng kí certificate cho mỗi server hoặc client.</li>
</ul>
- OpenVPN hỗ trợ xác thực 2 chiều dựa trên các certificate, có nghĩa là client phải xác thực certificate của server và server phải xác thực certificate của client trước khi sự tin tưởng lẫn nhau được thiết lập.
- Cả server lẫn client sẽ xác thực lẫn nhau bằng bước xác minh đầu tiên rằng certificate hiện tại được thiết lập bởi CA chủ, và sau đó test thông tin ở trong certificate header như tên certificate hoặc loại certificate.
- 1 số đặc điểm về mô hình bảo mật của OpenVPN:
<ul>
	<li>Server chỉ cần biết certificate của nó mà không cần biết bất kì certificate của client nào mà nó connect tới.</li>
	<li>Server chỉ chấp nhận các client mà certificate được thiết lập bởi CA chủ.</li>
	<li>Nếu 1 private key được thỏa hiệp thì nó cũng có thể bị disabled bằng cách add thêm certificate của nó vào CRL(certificate revocation list).CRL cho phép các certificate đã được thỏa hiệp bị từ chối mà không cần xây dựng lại PKI.</li>
	<li>Server có thể thực thi các truy cập đặc biệt của client dựa trên việc nhúng thêm certificate fields như là Common Name.</li>
</ul>

<a name="2"></a>
### 2.Thiết lập CA chủ,certificate và keys
#### 2.1.Thiết lập CA chủ
- Để thiết lập PKI, chúng ta sẽ dùng easy-rsa, 1 set các scripts đi kèm trong bản OpenVPN 2.2.x.Nếu bạn dùng OpenVPN 2.3.x thì bạn cần download tại <a href="https://github.com/OpenVPN/easy-rsa">đây</a>.</p>
- Giờ edit file vars và set các tham số KEY_COUNTRY, KEY_PROVINCE, KEY_CITY, KEY_ORG, and KEY_EMAIL.Không để trống bất kì tham số nào.
- Sau đó khởi tạo PKI
```sh
. ./vars
./clean-all
./build-ca
```
- Câu lệnh cuối `build-ca` sẽ tạo ra chứng chỉ CA và key.

#### 2.2.Thiết lập certificate và key cho server
- Tiếp theo chúng ta thiết lập certificate và private key cho server.
```sh
./build-key-server server		#server: tên tùy chỉnh bạn muốn đặt
```
- Trong các bước sau đó, hầu hết các tham số đã được đặt mặc định bạn chỉ việc "enter" và chọn yes khi được hỏi.

#### 2.3.Thiết lập certificate và keys cho client
- Tương tự như trên:
```sh
./build-key client1			
```
- Trong các bước sau đó, hầu hết các tham số đã được đặt mặc định bạn chỉ việc "enter" và chọn yes khi được hỏi.Đặc biệt phần Common Name, khi bạn đặt cho mỗi client thì nó phải là duy nhất.

#### 2.4.Thiết lập tham số Diffie Hellman
- Tham số Diffie Hellman phải được thiết lập cho OpenVPN server.
```sh
./build-dh
```

#### 2.5.Các file key
Filename | Needed by | Mục đích | Secret |
--- | --- | --- | --- |
ca.crt	| server + all clients	| Root CA certificate	| NO | 
ca.key	| key signing machine only	| Root CA key	| YES |
dh{n}.pem	| server only	| Diffie Hellman parameters	| NO |
server.crt	| server only	| Server Certificate	| NO |
server.key	| server only	| Server Key	| YES |
client1.crt	| client1 only	| Client1 Certificate	| NO |
client1.key	| client1 only	| Client1 Key	| YES |

- Bước cuối cùng là copy tất cả các file cần thiết tới các client.Đảm bảo copy các file cần bảo mật trên kênh truyền có bảo mật.

<a name="3"></a>
### 3.Tham Khảo
- https://openvpn.net/index.php/open-source/documentation/howto.html#dhcp
###
