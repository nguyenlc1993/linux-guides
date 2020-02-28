# Hướng dẫn đăng kí VPN Seed4Me miễn phí 1 năm
Link bài gốc:
- https://tinhte.vn/thread/vpn-mien-phi-1-nam-moi-nhat-2020.3060305/
- https://www.techno360.in/seed4-vpn-free-one-year/
- https://seed4.me/pages/setup/vpn-ubuntu-l2tp

## 1. Đăng kí tài khoản Seed4Me
1. Truy cập https://seed4.me/users/register và đăng kí tài khoản
2. Nhập code **CHOLLOMETRO** để  miễn phí 1 năm sử dụng
3. Sau khi đăng kí thì vào mail để confirm
4. Với Windows/Mac/iOS/Android thì truy cập vào https://seed4.me/pages/download để tải ứng dụng về, sau đó đăng nhập, chọn vùng & kết nối

## 2. Thiết lập VPN Seed4Me trên Linux
Ở đây sẽ setup VPN theo phương thức LT2P (do phương thức PPTP không hoạt động).

1. Cài đặt các gói cần thiết:
```sh
sudo add-apt-repository ppa:nm-l2tp/network-manager-l2tp
sudo apt update
sudo apt install network-manager-l2tp network-manager-l2tp-gnome
```
2. Vào **Settings/Network**, click dấu **+** ở tab VPN để add VPN
3. Chọn VPN connection type là **Layer 2 Tunneling Protocol (L2TP)**
4. Thiết lập VPN như sau:
    - Name: `sg.seed4.me` (có thể đặt tuỳ ý)
    - Gateway: IP của VPN server. Truy cập vào https://seed4.me/users/status#available_servers để lấy IP, nên chọn Singapore vì gần Việt Nam.
    - User name: Email đăng nhập Seed4Me
    - Password: Click vào icon bên cạnh, chọn **Store the password for all users**, sau đó nhập mật khẩu đăng nhập Seed4Me
5. Click nút **IPSec Settings...**
    - Check **Enable IPsec tunnel to LT2P host**
    - Nhập **Pre-shared key** là `seed4me`
    - Click OK
6. Click nút **Add** để thêm VPN, sau đó click vào switch để bật VPN
7. Khi hiện ra icon chìa khoá trên notification thì tức là VPN kết nối thành công.
8. Nếu gặp lỗi không kết nối được VPN, có thể xem log bằng lệnh:
```sh
journalctl --unit=NetworkManager
```