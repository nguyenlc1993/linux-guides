# Hướng dẫn cài đặt Ubuntu 18.04 trên Lenovo Legion Y530

## 1. Cấu hình
- CPU: Intel Core i7 8750H
- RAM: 16GB DDR4
- GPU: Intel UHD Graphics 630 + NVIDIA GeForce GTX 1050Ti
- Storage: 512 GB NVMe SSD + 2 TB HDD
- LAN: Realtek RTL8111
- Wifi: Realtek RTL8822BE -> Broadcom BCM94352z (DW1560)

## 2. Hướng dẫn
Guide gốc: https://github.com/kfechter/LegionY530Ubuntu/tree/18.04.2-Install-Guide

### 2.1. Lưu ý
Một số phím không hoạt động:
- Phím quay màn hình
- Phím chuyển màn hình (Fn + F7). Win + P vẫn xài được bình thường.
- Phím disable camera (Fn + F10)
- Phím disable microphone (Fn + F4)

Nếu máy bị treo thì có thể restart bằng Magic SysRq như sau:
1. Giữ Right Alt + PrtSc
2. Ấn lần lượt các phím R, E, I, S, U, B

### 2.2. Hướng dẫn cài
Ở đây sẽ dùng distro Pop!_OS dựa trên Ubuntu, do distro này có thể chuyển đổi giữa GPU NVIDIA và Intel dễ dàng. Tuy vậy cách cài với Ubuntu cũng tương tự.

1. Tải Pop!_OS 18.04 phiên bản NVIDIA tại https://system76.com/pop. Hoặc tải Ubuntu 18.04.2 tại https://ubuntu.com/download/desktop.
2. Ghi file ISO ra USB bằng Rufus (DD mode) hoặc Etcher.
3. Khởi động lại máy, bấm F2 liên tục để vào BIOS Setting. Vào tab Security và disable Secure Boot, sau đó bấm F10 và chọn Yes để lưu thiết lập.
4. Khởi động lại máy, bấm F12 liên tục để mở Boot menu. Chọn USB chứa bộ cài.
5. Bấm phím E để nhập boot option. Tìm dòng có chữ `quiet splash`, thêm `nouveau.modeset=0` và bấm F10 -> Disable nouveau driver khi boot để tránh máy bị treo sau khi login. **Lưu ý:** Với Pop!_OS do đã blacklist nouveau driver nên không cần.
6. Để bật Wifi, chạy lệnh:
```sh
sudo rmmod ideapad_laptop
```
7. Cài đặt Pop!_OS hoặc Ubuntu theo mode UEFI.
8. Khi khởi động lại và ở GRUB menu, làm tương tự bước 4 để boot vào OS.
9. Sau khi boot vào, chạy lần lượt các lệnh sau để update OS và các package lên mới nhất:
```sh
sudo apt update
sudo apt upgrade
sudo apt dist-upgrade
```

### 2.3. Cài NVIDIA driver
Phần này áp dụng với Ubuntu, còn Pop!_OS thì không cần.
1. Mở Software & Updates.
2. Vào tab Additional Drivers, chọn cài nvidia-driver-418 và bấm Apply Changes.
3. Khởi động lại máy, kiểm tra driver đã nhận bằng cách chạy lệnh `prime-select query`, nếu thấy output là `nvidia` thì là OK.
4. Để chọn GPU thì có thể dùng 1 trong 2 cách sau:
    - Vào NVIDIA X Server Settings, mở tab PRIME Profiles, chọn Intel hoặc NVIDIA, bấm Quit rồi khởi động lại máy.
    - Chạy lệnh `sudo prime-select <gpu>` trong đó `<gpu>` đặt là `intel` hoặc `nvidia`, sau đó khởi động lại máy.


### 2.4. Xử lý vấn đề wifi
#### 2.4.1. Wifi không hoạt động
Để wifi hoạt động thì phải blacklist module `ideapad_laptop` do nó gây conflict với wifi. Cách làm như sau:
1. Chạy lệnh `sudo gedit /etc/modprobe.d/blacklist.conf`.
2. Bổ sung thêm dòng sau vào cuối file.
```sh
# Enable wifi working on Y530
blacklist ideapad_laptop
```
3. Lưu file lại và chạy câu lệnh sau để update boot image:
```sh
sudo update-initramfs -u
```
4. Khởi động lại và kiểm tra Wifi đã bật.

**Lưu ý:** Việc disable module này sẽ khiến các hotkey liệt kê ở phần **2.1** không hoạt động. Việc upgrade kernel lên version 4.20 có thể sẽ khắc phục được lỗi này.

#### 2.4.2. Với card Realtek RTL8822BE
Chạy các lệnh sau để enable wifi RTL8822BE:

```sh
echo "options r8822be aspm=0" | sudo tee /etc/modprobe.d/r8822be.conf
sudo rmmod r8822be
sudo modprobe r8822be
```

Chi tiết bug: https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1806472

#### 2.4.3. Với card Broadcom BCM94352z
Để lấy ID card Wifi, ta chạy lệnh:
```sh
lspci -nn | grep Network
```
Ví dụ output:
```
07:00.0 Network controller [0280]: Broadcom Inc. and subsidiaries BCM4352 802.11ac Wireless Network Adapter [14e4:43b1] (rev 03)
```
Để lấy ID card Bluetooth, ta chạy lệnh:
```sh
lsusb
```
Ví dụ output:
```
Bus 001 Device 004: ID 0489:e07a Foxconn / Hon Hai 
```

Chạy lệnh sau để enable wifi BCM94352z:
```sh
sudo apt update
sudo apt install --reinstall broadcom-sta-dkms
sudo modprobe -r b43 ssb wl brcmfmac brcmsmac bcma
sudo modprobe wl
```

Để cài & load bluetooth firmware, ta làm như sau:
1. Tải firmware ứng với ID của card bluetooth tại https://github.com/winterheart/broadcom-bt-firmware (đuôi `.hcd`).
2. Copy file firmware vào `/lib/firmware/brcm`.
3. Chạy các lệnh sau để reload module bluetooth:
```sh
sudo modprobe -r btusb
sudo modprobe btusb
```

Source:
- https://help.ubuntu.com/community/WifiDocs/Driver/bcm43xx#STA_-_Internet_access
- https://unix.stackexchange.com/questions/359979/broadcom-bcm4352-bluetooth-does-not-connect
- https://dev-pages.info/ubuntu-bluetooth/

### 2.5. Cấu hình combo audio jack
1. Mở file alsa-base.conf bằng lệnh:
```sh
sudo gedit /etc/modprobe.d/alsa-base.conf
```
2. Thêm dòng sau vào cuối file:
```sh
# Configure audio jack to work with microphones
options snd-hda-intel model=dell-headset-multi
```
3. Khởi động lại máy và kiểm tra.

### 2.6. Set default brightness khi khởi động
Mặc định khi khởi động thì brightness set ở giá trị 50%. Để set giá trị khác ta làm như sau:
1. Chỉnh brightness lên mức mà ta muốn.
2. Chạy câu lệnh sau để lấy giá trị brightness tương ứng:
```sh
cat /sys/class/backlight/intel_backlight/brightness
```
Ví dụ output sẽ là `120000` với brightness 100%.

3. Tạo file `/etc/rc.local` với lệnh:
```sh
sudo gedit /etc/rc.local
```
4. Paste nội dung sau vào file và lưu lại:
```sh
#!/bin/bash
echo 108000 > /sys/class/backlight/intel_backlight/brightness
exit 0
```
5. Khởi động lại máy và kiểm tra

### 2.7. Các vấn đề khác
Để hỗ trợ multitouch, ta cài thư viện `libinput-gestures` tại https://github.com/bulletmark/libinput-gestures.

Để enable nút Airplane, ta có thể dùng plugin Xmodmap.

Chi tiết xem tại https://github.com/kfechter/LegionY530Ubuntu/blob/master/Sections/Advanced.md
