# Giới thiệu
Vagrant có thể hiểu đơn giản là một tool giúp tạo máy ảo để dev một cách đơn giản và nhanh chóng. Vagrant sử dụng các phần mềm máy ảo có sẵn như VirtualBox, VMWare, Hyper-V, Docker, v.v. và dùng các box được host trên repo của nó để tạo máy ảo.

Một số Vagrant box phổ biến nhất:
- laravel/homestead: Dùng để code Laravel
- centos/7: CentOS 7
- ubuntu/bionic64: Ubuntu 18.04

Vagrant box có thể được tìm kiếm tại [đây](https://app.vagrantup.com/boxes/search).

# Cài đặt Vagrant
## 1. Cài VirtualBox
Follow hướng dẫn tại [đây](https://www.virtualbox.org/wiki/Linux_Downloads).

Cài VirtualBox 6.0 trên Ubuntu 18.04:

```sh
wget -q https://www.virtualbox.org/download/oracle_vbox_2016.asc -O- | sudo apt-key add -
wget -q https://www.virtualbox.org/download/oracle_vbox.asc -O- | sudo apt-key add -
sudo add-apt-repository "deb http://download.virtualbox.org/virtualbox/debian bionic contrib"
sudo apt update
sudo apt install virtualbox-6.0
```

Cài thêm Extension Pack (tuỳ chọn) ở [đây](https://download.virtualbox.org/virtualbox/6.0.12/Oracle_VM_VirtualBox_Extension_Pack-6.0.12.vbox-extpack).

## 2. Cài Vagrant
Cài Vagrant trên Ubuntu 18.04:

```sh
sudo apt install vagrant
```

Kiểm tra Vagrant đã cài đặt:

```sh
vagrant --version
```

Cài plugin `vagrant-vbguest` để tự động cài VirtualBox Guest Additions lên VM.

```sh
vagrant plugin install vagrant-vbguest
```

Các plugins khác cài thêm ở [đây](https://github.com/hashicorp/vagrant/wiki/Available-Vagrant-Plugins).


# Sử dụng Vagrant
## 1. Các lệnh cơ bản
- `vagrant init <box>` - Khởi tạo Vagrantfile. Ví dụ: `vagrant init centos/7`.
- `vagrant up` - Tạo máy ảo (nếu chưa có) và khởi động.
- `vagrant suspend` - Tạm ngưng máy ảo.
- `vagrant resume` - Đánh thức máy ảo.
- `vagrant halt` - Tắt máy ảo.
- `vagrant destroy` - Tắt và xoá máy ảo khỏi bộ nhớ.
- `vagrant reload` - Khởi động lại máy ảo và load lại từ Vagrantfile.
- `vagrant ssh` - Kết nối SSH vào máy ảo.
- `vagrant snapshot push` - Tạo snapshop cho máy ảo.
- `vagrant snapshot pop` - Khôi phục lại snapshot gần nhất.
- `vagrant provision` - Chạy script provision để cài đặt phần mềm.

## 2. Cấu hình máy ảo
Với provider là VirtualBox, ta thêm đoạn sau vào Vagrantfile:

```
config.vm.provider "virtualbox" do |vb|
    # VM name
    vb.name = "CentOS 7"
    # Enable/disable GUI
    vb.gui = true
    # Memory size (MB)
    vb.memory = 2048
    # Number of CPU cores
    vb.cpus = 2
end
```

## 3. Cài đặt GUI cho CentOS
Link gốc: https://codingbee.net/vagrant/vagrant-enabling-a-centos-vms-gui-mode

Thêm dòng sau vào Vagrant file:
```
config.vm.provision "shell", path: "https://gist.githubusercontent.com/Sher-Chowdhury/0f96ee71fcec5b3b5c205bc277a93f6b/raw/2b033ed20d6fe986449e8019eca884cbaba35f2d/install-gnome-gui-on-centos7.sh"
```

Chạy `vagrant provision` để setup.
