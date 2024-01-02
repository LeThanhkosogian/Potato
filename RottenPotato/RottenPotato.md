# Rotten Potato🐛
## Hot Potato sử dụng một số kĩ thuật phức tạp như giả mạo NBNS, WPAD và Windows Update để lừa Windows xác thực với chúng tôi qua HTTP. Tiếp theo, chúng ta sẽ thảo luận về một phương pháp khác để đạt được mục đích tương tự, đó là Rotten Potato do Stephen Breen [@breenmachine](https://twitter.com/breenmachine) công bố. 
## Về cơ bản, Rotten sẽ lừa DCOM/RPC để xác thực NTLM. Ưu điểm của phương pháp phức tạp hơn này là nó đáng tin cậy 100%, nhất quán trên các phiên bản Windows và tính tức thời thay vì đôi khi phải đợi Windows Update như Hot Potato.
## Luồng hoạt động của Rotten Potato
![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/3b4ee42e-31de-4a99-bdbd-24d1eff91ea6)
### Dù tinh vi hơn Hot Potato, song Rotten Potato vẫn được chia làm 3 phần chính. 
## 1. Local NBNS Spoofer: Attacker mạo danh tên phân giải từ địa chỉ IP để khiến cho Victim tải file cấu hình WPAD độc hại.
### 1.1. NBNS (WINS) (port 137) (NetBIOS Name Service)
   - NBNS là giao thức quảng bá (broadcast) hoạt động trên nền UPD phục vụ viện phân giải tên, thường được sử dụng trong môi trường Windows.
   - Khi thực hiện một DNS lookup:
     - Trước tiên, Windows sẽ kiểm tra file "hosts" trong "C:\Windows\System32\drivers\etc".
     - Nếu không có, Windows thực hiện DNS lookup để tìm.
     - Nếu không thể tìm thấy, Windows sẽ thực hiện NBNS lookup. Giao thức NBNS sẽ hỏi tất cả các host có trong mạng nội bộ bằng cách truyền Broadcast "Who knows the IP address for host XXX?". Bất kể một host nào trong mạng đều có thể tự do trả lời gói tin này.
### 1.2. NBNS in Rotten Potato:
   - Lợi dụng điểm yếu của NBNS khi tất cả các host đều có thể trả lời gói tin broadcast hỏi địa chỉ, Attacker có thể đánh lừa hệ thống của Victim rằng Attacker chính là nơi mà Victim đang tìm.
   ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/bbca36e6-3457-4570-8c6c-3d7b89340252)
   - Attacker nghe lén mọi lưu lượng mạng và phản hồi tất cả các truy vấn NBNS trong mạng nội bộ, cố gắng giả mạo mọi hosts, trả lời các requests bằng địa chỉ IP của Attacker. Thế nhưng, thực tế thì Attacker không thể có quyền nghe lén tất cả lưu lượng mạng (điều này đòi hỏi phải có Local administrator). Vậy làm thế nào để Attacker có thể thực hiện NBNS spoofing?
      - Trong thực tế, Attacker có thể biết trước được tên host mà Victim muốn tìm kiếm. Attacker có thể làm giả response (gửi lại cho Victim) và cho dù có TXID (Transaction ID) để xác minh nhưng đó chỉ là số hexa 16 bit (và vì là giao thức UDP nên việc trao đổi sẽ rất nhanh), Attacker lại một lần nữa có thể dùng kĩ thuật Flood để thử nhiều nhất là 2^16=65536 khả năng để đánh lừa Victim.
        ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/8edc2292-ed4a-4c5f-9b6b-f227490c9f0e)
      - Thế nhưng sẽ thế nào nếu lỡ trong mạng nội bộ đã có sẵn bản ghi DNS mà Victim đang muốn tìm ? Attacker có thể sử dụng kĩ thuật gây cạn kiệt các UDP port, khiến cho mọi DNS lookups đều thất bại => Buộc Victim phải dùng NBNS.
## 2. Fake WPAD Proxy Server: Attacker triển khai file cấu hình WPAD độc hại để buộc Victim phải thực hiện xác thực NTLM.
### 2.1. WPAD (Web Proxy Auto Discovey)
   - Là giao thức tự động phát hiện máy chủ proxy cho các yêu cầu HTTP. Được sử dụng bởi các trình duyệt web và các ứng dụng khác để xác định máy chủ proxy mà chúng cần sử dụng để truy cập các trang web và tài nguyên web.
   - WPAD hoạt động bằng cách sử dụng một tệp văn bản có tên là "WPAD.dat". Tệp này được lưu trữ trên máy chủ proxy hoặc trên một máy chủ khác trong mạng. Tệp này chứa thông tin về máy chủ proxy, chẳng hạn như địa chỉ IP, cổng và tên miền.
   - Khi một ứng dụng cần truy cập một trang web, nó sẽ gửi một yêu cầu đến máy chủ proxy. Yêu cầu này sẽ bao gồm địa chỉ IP của trang web mà ứng dụng đang cố gắng truy cập. Máy chủ proxy sẽ trả lời yêu cầu bằng cách gửi tệp WPAD.dat. Ứng dụng sẽ sử dụng thông tin trong tệp này để cấu hình chính nó để sử dụng máy chủ proxy.
### 2.2. WPAD in Rotten Potato
   - Là giao thức tạo ra với mục đích tốt, tăng tính tiện lợi cho người dùng nhưng có thể bị Attacker lạm dụng. Cụ thể, sau khi đẫ NBNS Spoofing thành công, Victim đã ngỡ Attacker là WPAD-"người em luôn tìm kiếm", Attacker sẽ cấu hình một tệp WPAD.dat độc hại để chỉ định cho Victim rằng máy chủ proxy của Attacker là máy chủ cập nhật Windows. Khi một người dùng cập nhật Windows, máy họ sẽ được cấu hình để sử dụng proxy của Attacker.
     ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/4190058a-652c-4cf9-b4fb-9fc1fdb29e86)
   - Khi người dùng thường tải xuống bản cập nhật, họ sẽ được yêu cầu xác thực NTML với máy chủ proxy. Attacker sẽ sử dụng kĩ thuật NTML relay để tận dụng token xác thực này.
## 3. HTTP -> SMB NTLM Relay: Attacker sử dụng WPAD NTML token để truy cập SMB và tạo ra tiển trình có đặc quyền.
### 3.1. NTLM (Windows New Technology LAN Manager)
   - Là giao thức xác thực dạng Challenge/Response (Thử thách / Phản hồi)
   - SSO (Single Sign-On): đăng nhập 1 lần
   - Là giao thức khá lỗi thời (bị Kerberos thay thế), nhưng vẫn được sử dụng đến nay (2024) vì tương thích với hệ thống cũ
      ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/99b72562-f65f-4201-a131-fe7a096af13e)
   - Hoạt động theo cơ chế Three-way Handshake
      - NEGOTIATE: thông điệp yêu cầu Trao đổi từ Client
      - CHALLENGE: thông điệp Thử thách từ Server
      - AUTHENTICATE: thông điệp xác thực từ Client
         ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/d105fcae-6e40-4c5a-90f5-b06e8ca40a91)
   - Cụ thể hơn:
      - Client gửi bản text chứa Username đến Server
      - Server gửi cho Client 1 "Đề": 16 byte random number
      - Client dùng Pwd/NTLMHashedPwd mã hoá "Đề" rồi gửi Server
      - Server lại gửi "Đề thi", "Lời giải" của Client và Username đến DC (Domain Controller)
      - DC tìm Username trong DC rồi dùng Pwd/NTLMHashedPwd để "Giải đề"
      - Nếu "Lời giải" của DC và Client trùng nhau -> OK
         ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/680f19e4-d5cd-453f-9ca6-0fd6cee57999)
#### 3.2. NTML Relay in Rotten Potato
   - Sau khi đánh lừa Victim sử dụng proxy của mình làm Server cập nhật Windows thì việc của Attacker chỉ cần ngồi chờ một người dùng nào đó thực hiện việc xác thực NTLM trên hệ thống Attacker để tiến hành NTML Relay.
   - NTML Relay có nhiều biến thể khác nhau, một phiên bản cũ hơn là SMB -> SMB NTML Relay. Attacker sẽ lừa người dùng xác thực NTML vào SMB service của mình và sử dụng chính token có được để xác thực ngược lại máy của người dùng (cũng qua giao thức SMB). Thế nhưng, bản vá của Windows đã cấm việc sử dụng xác thực NTML cũng một giao thức với cũng một thử thách đang được tạo ra (vì chẳng có ai đã có quyền truy cập SMB lại đi đòi xác thực NTML để một lần nữa truy cập SMB cả) => SMB -> SMB NTML Relay không còn dùng được. Dù thế nào đi nữa, hầu hết các cuộc tấn công Relaying vẫn thường theo dạng MITM như hình sau:
     ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/be9453af-2204-49df-88ac-007ced8e1195)
   - Bất chấp việc không thể dùng SMB -> SMB NTML Relay, Attacker lại một lần nữa sáng tạo hơn bằng cách sử dùng HTTP -> SMB Relay. Mọi xác thực NTML để có thể tải xuống file cập nhật Windows của Victim sẽ là dạng HTTP và Attacker sẽ sử dụng nó để xác thực SMB services (hoặc bất kì một dịch vụ nào khác cần NTML xác thực). Tuỳ vào người dùng nào đã xác thực HTTP NTLM thì Attacker sẽ nhận được quyền tương ứng, nếu đó là Windows Update service thì câu lệnh sẽ được chạy quyền "NT AUTHORITY SYSTEM".

## Is Patched?
   - Microsoft đã phát hành bản vá MS16-075 không cho phép xác thực NTLM cũng giao thức bằng một Challenge đã có sẵn (tức là SMB -> SMB NTLM Relay từ một host trở lại chính nó như đã đề cập bên trên).
   - MS16-077 WPAD Name Resolution sẽ không sử dụng NetBIOS (CVE-2016-3213) và không được phép gửi thông tin xác thực khi yêu cầu tệp PAC (CVE-2016-3236) => WPAD MITM đã được vá.

## References
[https://foxglovesecurity.com/2016/01/16/Rotten-potato/](https://foxglovesecurity.com/2016/01/16/Rotten-potato/)

[Binary Repo](https://github.com/foxglovesec/Potato)
