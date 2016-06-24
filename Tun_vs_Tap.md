- TUN và TAP là 2 thiết bị mạng ảo chạy trong kernel.
- TUN (Tunnel) như 1 thiết bị lớp mạng, nó làm việc với các gói tin lớp 3 như là gói tin IP.
- TAP được như 1 thiết bị lớp data link, nó làm việc với các gói tin lớp 2 như là  ethernet frames.
- TUN được sử dụng bằng cách routing, TAP được sử dụng bằng cách tạo ra network bridge.
- TAP được sử dụng trong các trường hợp:
<ul>
  <li>Khi ta muốn transport các traffic không dựa trên nền IP.</li>
  <li>Khi ta muốn mạng lan của ta và VPN clients trong cùng 1 broadcast domain.</li>
  <li>Khi ta muốn DHCP server trong lan cấp ip cho VPN clients.</li>
  <li></li>
</ul>
- TUN được sử dụng trong các trường hợp:Khi ta muốn tạo 1 subnet riêng giữa 2 đầu, và chỉ chạy trên nền IP.</li>
- Ưu điểm của TAP:
<ul>
  <li>Có thể transport bất kì protocol nào (IPv4,IPv6, Netalk, IPX ...)</li>
  <li>Có thể sử dụng trên bridge</li>
</ul>
- Nhược điểm của TAP:
<ul>
  <li>Là nguyên nhân tạo ra nhiều broadcast trong VPN tunnel.</li>
  <li>Add thêm Ethernet header trong mọi gói tin được chuyển qua VPN tunnel.</li>
  <li>Khả năng mở rộng kém.</li>
  <li>Không thể sử dụng các thiết bị iOS, Android.</li>
</ul>
- Ưu điểm của TUN:
<ul>
  <li>Ít traffic đi qua tunnel hơn, chỉ transport các traffic có đích đến là VPN client.</li>
  <li>Chỉ transport các gói tin IP lớp 3.</li>
</ul>
- Nhược điểm của TUN:
<ul>
  <li>Các traffic broadcast không được transport.</li>
  <li>Chỉ có thể transport IPV4 (bản OpenVPN 2.4 hỗ trợ IPv6)</li>
  <li>Không thể sử dụng trên bridge.</li>
</ul>
