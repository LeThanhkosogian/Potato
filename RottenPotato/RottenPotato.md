# Rotten Potato🐛

## Hot Potato sử dụng một số kĩ thuật phức tạp như giả mạo NBNS, WPAD và Windows Update để lừa Windows xác thực với chúng tôi qua HTTP. Tiếp theo, chúng ta sẽ thảo luận về một phương pháp khác để đạt được mục đích tương tự, đó là Rotten Potato do Stephen Breen [@breenmachine](https://twitter.com/breenmachine) công bố.

## Về cơ bản, Rotten sẽ lừa DCOM/RPC để xác thực NTLM. Ưu điểm của phương pháp phức tạp hơn này là nó đáng tin cậy 100%, nhất quán trên các phiên bản Windows và tính tức thời thay vì đôi khi phải đợi Windows Update như Hot Potato.

## Luồng hoạt động của Rotten Potato
![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/3b4ee42e-31de-4a99-bdbd-24d1eff91ea6)

## Các bước được mô tả như sau:
1. Điều hướng RPC để yêu cầu xác thực với proxy:
- Sử dụng lệnh gọi API CoGetInstanceFromIStorage.
- Chỉ định địa chỉ IP và cổng của proxy trong lệnh gọi này (127.0.0.1:6666).
2. RPC gửi gói NTLM Negotiate đến proxy.
3. Proxy chuyển tiếp gói NTLM Negotiate đến RPC ở cổng 135:
- Sử dụng gói này làm mẫu.
- Đồng thời, gọi AcceptSecurityContext để bắt buộc xác thực local.
- Lưu ý: Gói này được sửa đổi để bắt buộc xác thực local.
4. & 5. RPC 135 và AcceptSecurityContext phản hồi với gói NTLM Challenge:
- Nội dung của cả hai gói được mix để khớp với local negotiation.
- Gói tin được chuyển tiếp đến RPC (bước 6).
6. Proxy chuyển tiếp gói NTLM Challenge đã mix đến RPC.
7. RPC phản hồi với gói NTLM Auth:
- Gói này được gửi đến AcceptSecurityContext (bước 8).
8. Gói NTLM Auth được gửi đến AcceptSecurityContext.
9. Thực hiện mạo danh (impersonation).

### Dù tinh vi hơn Hot Potato, song Rotten Potato vẫn được chia làm 3 phần chính.

## Phần 1. Trick "NT AUTHORITY\SYSTEM" into authenticating NTLM
![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/1de1b5ee-ba71-4ca2-b7a3-bd2d0918f639)
**Lợi dụng việc RPC chạy dưới quyền "NT AUTHORITY/SYSTEM" sẽ cố xác thực local proxy (TCP endpoint) của Attacker qua lệnh gọi API "CoGetInstanceFromIStorage".**
### 1.1. RPC (Remote Procedure Call)
- Là giao thức mạng, mô hình Client-Server, thực hiện giao tiếp giữa các tiến trình.
- Hiểu đơn giản là phương pháp gọi hàm (dịch vụ) từ một máy tính ở xa để lấy về kết quả.
- Ngoài ra còn có gRPC, giúp thực hiện giao tiếp giữa các máy.
- Mô hình hoạt động của RPC:
  
![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/b85e48c3-b73e-441d-9958-5ecdf16e7158)
### 1.2. COM (Component Object Mode) và CoGetInstanceFromIStorage API
- COM là một mô hình đối tượng được phát triển bởi Microsoft, cung cấp interfaces và API để các thành phần COM tương tác với nhau.
- COM cho phép liên lạc giữa các thành phần phần mềm của các ngôn ngữ khác nhau.
- "CoGetInstanceFromIStorage" được sử dụng để tạo một đối tượng COM từ một đối tượng IStorage đã tồn tại.
- Ngoài ra còn có DCOM (Distribute COM).
### 1.3. RPC & COM in Rotten Potato
- Bắt đầu, Attacker sẽ lạm dụng "CoGetInstanceFromIStorage" API gọi đến COM, quá trình được mô tả trong đoạn code mẫu sau:
![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/041bb2a1-3ce6-48f2-9881-15bcb47df599)
  - Khởi tạo Method BootstrapComMarshal(), kiểu trả về **void**, độ truy cập **public**, thuộc về Class **static**
  - Trong phương thức:
    - Tạo đối tượng IStorage **stg** để lưu trữ dữ liệu liên quan đến COM bằng cách gọi hàm **ComUtils.CreateStorage()**
    - Tạo Guid (Global ID) **clsid** đại diện cho một máy chủ COM đã biết trên local, cụ thể là BITSv1
    - Tạo đối tượng TestClass (Là một lớp custom của Attacker) **c**, truyền **stg** và tạo một chuỗi chứa địa chỉ IP:port (127.0.0.1:6666)
    - Chuẩn bị mảng MULTI_QI để yêu cầu interfaces IUnknownPtr
    - Gọi hàm CoGetInstanceFromIStorage() để khởi tạo đối tượng COM, truyền các đối số sau
      - null: Không có tệp lưu trữ cụ thể
      - ref clsid: Tham chiếu đến Guid của máy chủ COM
      - null: Không có thuộc tính tùy chỉnh
      - CLSCTX.CLSCTX_LOCAL_SERVER: Chỉ định tìm máy chủ COM trên hệ thống local
      - c: Đối tượng TestClass đã tạo
      - 1: Số lượng interfaces được yêu cầu
      - qis: Mảng MULTI_QI chứa thông tin về các interfaces được yêu cầu
## Phần 2. Man-in-the-middle (MITM)
![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/3542fb68-2bd5-436b-ac8d-ce2c1cc2ab2b)
**RPC port 135 bị lạm dụng để làm template, giúp Attacker trả lời tất cả các request từ First RPC (bên phải) thực hiện**
### 2.1. Hunting NTLM
- Đến bước hiện tại, Attacker đã có một COM giao tiếp với local TCP listener của Attacker trên port 6666. Việc cần làm bây giờ là Attacker cần phải giao tiếp với COM (đang chạy dưới quyền NT AUTHORITY/SYSTEM) sao cho đúng để có thể xác thực NTLM.
- COM sẽ trao đổi bằng giao thức RPC, rất khó để Attacker có thể nắm rõ giao thức RPC theo từng phiên bản Windows. Vì vậy, Attacker sử dụng một kĩ thuật khác để tạo ra câu trả lời chính xác:
  - Attacker sẽ forward tất cả các gói tin từ COM trên TCP 6666 quay trở lại local Windows RPC port 135
  - Sử dụng gói tin nhận được từ RPC 135 để làm template (mẫu) để giao tiếp với COM
- Cụ thể hơn:
  ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/da7b4f1b-1c94-4866-9427-946edb6305f5)
  - Packet 7 (127.0.0.1:49247-COM to 127.0.0.1:6666-Attacker), COM đang trao đổi với TCP listener 6666 của Attacker
  - Packet 9 (127.0.0.1:49248 to 127.0.0.1:135-TempRPC), Attacker chuyển tiếp giống như Packet 7 tới RPC TCP 135
  - Packet 11 (127.0.0.1:135-TempRPC to 127.0.0.1:49248), Attacker nhận phản hồi từ RPC TCP 135
  - Packet 13 (127.0.0.1:6666-Attacker to 127.0.0.1:49247-COM), Attacker chuyển tiếp phản hồi từ Packet 11 đến COM
  - Lặp đi lặp lại quá trình này => cho đến khi NTLM diễn ra
### 2.2. NTLM Relay at a high level
**NTLM Relay của Rotten Potato phức tạp hơn nhiều so với Hot Potato**
- Trước tiên, cần làm rõ quy trình dưới đây:
![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/f885f989-fb71-4699-85fe-8669df09f03e)
  - Bên trái, COM sẽ gửi NTLM Negotiate trên TCP 6666. Bên phải, Attacker gọi API Windows "AcquireCredentialsHandle"
  - Tiếp theo, Attacker gọi "AcceptSecurityContext" và truyền đầu vào là NTLM Type 1 (Negotiate), output sẽ là NTLM Type 2 (Challenge) và gửi lại cho Client yêu cầu xác thực (trong TH này là DCOM)
  - Khi Client (DCOM) phản hồi bằng NTLM Type 3 (Authenticate), chuyển nó sang lệnh gọi API "AcceptSecurityContext" => Hoàn tất xác thực và nhận được token
- Sau khi MITM để relay gói tin tại **2.1. Hunting NTML**, sẽ thầy COM bắt đầu xác thực NTLM:
   ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/f0d3eea0-33d0-4d88-a0db-e0dd8a6e304d)
 - **NTLM TYPE 1 (NEGOTIATE)**:
   - Packet 29 (127.0.0.1:49249-COM to 127.0.0.1:6666-Attacker), thông điệp yêu cầu trao đổi từ COM
   - Packet 31, Attacker relay đến RPC 135
 - **NTLM TYPE 2 (CHALLENGE)**:
   - Packet 33 (127.0.0.1:135-TempRPC to 127.0.0.1:6666-49250), RPC 135 sẽ phản hồi bằng gói NTLM Challenge, đây là hình ảnh 2 gói NTLM Type 2, trái là TempRPC->Attacker, phải là Attacker->COM:
     ![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/abdec887-c01a-4a02-8ca5-9156d9ce0bcc)
   - Thay vì lập tức chuyển tiếp gói này đến COM, Attacker thực hiện 1 số thao tác, thay đổi giá trị phần **"NTLM Server Challenge" và "Reserved"** để đảm bảo lời gọi API "AcceptSecurityContext" sẽ thành công. Nếu không, RPC 135 sẽ được xác thực thay vì Attacker.
   - Sau khi sửa đổi, chuyển tiếp gói NTLM TYPE 2 (CHALLENGE) đến COM
- **NTLM TYPE 3 (AUTHENTICATE)**:
  - COM dưới quyền NT AUTHORITY/SYSTEM sẽ gửi lại NTLM TYPE 3 (AUTHENTICATE) => Dùng nó để thực hiện lệnh gọi cuối đến "AcceptSecurityContext"
  - Sau đó, gọi “ImpersonateSecurityContext” để nhận Impersonation Token


## Phần 3. Impersonate token
Gọi API "AcceptSecurityContext" để mạo danh "NT AUTHORITY/SYSTEM". Việc mạo danh (impersonate) chỉ có thể thành công nếu Attacker đang chiếm được TK người dùng có quyền (impersonate security token). Quyền cần thiết này thường thấy ở các TK Service (như Web(IIS), Database(MSSQL),...), gần như không có ở TK người dùng thường.
- [Social Engineering the Windows Kernel - James Forshaw’s BlackHat](https://www.youtube.com/watch?v=QRpfvmMbDMg&ab_channel=BlackHat)
- [Rotten Background and Demo](https://www.youtube.com/watch?v=8Wjs__mWOKI&ab_channel=AdrianCrenshaw)
- [IIS Demo](https://youtu.be/wK0r-TZR7w8)
- [MSSQL Demo](https://youtu.be/3CPdKMeB0UY)

## Is Patched?
## References
- [RPC vs gRPC overview](https://viblo.asia/p/gioi-thieu-ve-rpc-va-grpc-E1XVOxOZ4Mz)
- [COM overview](https://www.youtube.com/watch?app=desktop&v=7FA3PKyg3Vo&ab_channel=CSEMA)
