# Rotten Potato🐛
## Hot Potato sử dụng một số kĩ thuật phức tạp như giả mạo NBNS, WPAD và Windows Update để lừa Windows xác thực với chúng tôi qua HTTP. Tiếp theo, chúng ta sẽ thảo luận về một phương pháp khác để đạt được mục đích tương tự, đó là Rotten Potato do Stephen Breen [@breenmachine](https://twitter.com/breenmachine) công bố. 
## Về cơ bản, Rotten sẽ lừa DCOM/RPC để xác thực NTLM. Ưu điểm của phương pháp phức tạp hơn này là nó đáng tin cậy 100%, nhất quán trên các phiên bản Windows và tính tức thời thay vì đôi khi phải đợi Windows Update như Hot Potato.
## Luồng hoạt động của Rotten Potato
![image](https://github.com/LeThanhkosogian/Potato/assets/97555997/3b4ee42e-31de-4a99-bdbd-24d1eff91ea6)
## Các bước được mô tả như sau:
1. Điều hướng RPC để yêu cầu xác thực với proxy:
- Sử dụng lệnh gọi API CoGetInstanceFromIStorage.
- Chỉ định địa chỉ IP và cổng của proxy trong lệnh gọi này.
2. RPC gửi gói NTLM Negotiate đến proxy.
3. Proxy chuyển tiếp gói NTLM Negotiate đến RPC ở cổng 135:
- Sử dụng gói này làm mẫu.
- Đồng thời, gọi AcceptSecurityContext để bắt buộc xác thực cục bộ.
- Lưu ý: Gói này được sửa đổi để bắt buộc xác thực cục bộ.
4. & 5. RPC 135 và AcceptSecurityContext phản hồi với gói NTLM Challenge:
- Nội dung của cả hai gói được trộn lẫn để khớp với một cuộc đàm phán cục bộ.
- Gói tin hỗn hợp được chuyển tiếp đến RPC (bước 6).
6. Proxy chuyển tiếp gói NTLM Challenge đã trộn lẫn đến RPC.
7. RPC phản hồi với gói NTLM Auth:
- Gói này được gửi đến AcceptSecurityContext (bước 8).
8. Gói NTLM Auth được gửi đến AcceptSecurityContext.
9. Thực hiện mạo danh (impersonation).
### Dù tinh vi hơn Hot Potato, song Rotten Potato vẫn được chia làm 3 phần chính. 
## Phần 1. Trick "NT AUTHORITY\SYSTEM" into authenticating NTLM
Lợi dụng việc RPC chạy dưới quyền 
### 1.1. 
### 1.2. 
## Phần 2. Man-in-the-middle (MITM)
### 2.1.
### 2.2.
## Phần 3. Impersonate token
### 3.1. 
### 3.2.
 

## Is Patched?
## References

