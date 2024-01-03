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
Lợi dụng việc RPC chạy dưới quyền "NT AUTHORITY/SYSTEM" sẽ cố xác thực local proxy (TCP endpoint) của Attacker qua lệnh gọi API "CoGetInstanceFromIStorage".
### 1.1. RPC (Remote Procedure Call)
- Là giao thức mạng, mô hình Client-Server, thực hiện giao tiếp giữa các tiến trình.
- Hiểu đơn giản là phương pháp gọi hàm (dịch vụ) từ một máy tính ở xa để lấy về kết quả.
- Ngoài ra còn có gRPC, giúp thực hiện giao tiếp giữa các máy.
- Mô hình hoạt động của RPC:
  
![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/b85e48c3-b73e-441d-9958-5ecdf16e7158)
### 1.2. COM (Component Object Mode) và CoGetInstanceFromIStorage API
- COM là một mô hình đối tượng được phát triển bởi Microsoft, cung cấp interfaces và API để các thành phần COM tương tác với nhau.
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
      - CLSCTX.CLSCTX_LOCAL_SERVER: Chỉ định tìm máy chủ COM trên hệ thống cục bộ
      - c: Đối tượng TestClass đã tạo
      - 1: Số lượng interfaces được yêu cầu
      - qis: Mảng MULTI_QI chứa thông tin về các interfaces được yêu cầu
## Phần 2. Man-in-the-middle (MITM)
RPC công 135 bị lạm dụng để làm mẫu, giúp Attacker trả lời tất cả các request First RPC (bên phải) thực hiện. 
### 2.1.
### 2.2.

## Phần 3. Impersonate token
Gọi API "AcceptSecurityContext" để mạo danh "NT AUTHORITY/SYSTEM". Việc mạo danh (impersonate) chỉ có thể thành công nếu Attacker đang chiếm được TK người dùng có quyền (impersonate security token). Quyền cần thiết này thường thấy ở các TK Service (như Web, Database,...), gần như không có ở TK người dùng thường.
### 3.1. 
### 3.2.

## Is Patched?
## References
- [RPC vs gRPC overview](https://viblo.asia/p/gioi-thieu-ve-rpc-va-grpc-E1XVOxOZ4Mz)
- [COM overview](https://www.youtube.com/watch?app=desktop&v=7FA3PKyg3Vo&ab_channel=CSEMA)
