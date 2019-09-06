# Thủ thuật khi dùng Ụbuntu
## 1. Đặt nhiệt độ màu cho Night Light
Chạy câu lệnh dưới đây để đặt nhiệt độ màu cho tính năng Night Light:
```
gsettings set org.gnome.settings-daemon.plugins.color night-light-temperature <temperature>
```
Giá trị tham khảo:
- 1000 Giá trị thấp nhất (siêu vàng)
- 4000 Giá trị default khi bật Night Light (vàng)
- 5500 Giá trị ứng với nhiệt độ nhẹ (vàng nhẹ)
- 6500 Giá trị default khi tắt Night Light (xanh nhẹ)
- 10000 Giá trị cao nhất (siêu xanh)

Có thể dùng Dconf Editor để setting cho dễ.

Source: https://askubuntu.com/questions/914500/how-to-adjust-the-hue-intensity-of-gnome-night-light
