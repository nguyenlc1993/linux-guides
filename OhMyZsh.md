# Hướng dẫn cài đặt Oh My Zsh
## 1. Yêu cầu
- OS: macOS hoặc Linux
- Đã cài zsh, curl, git
## 2. Cài Zsh
1. Mở Terminal, chạy lệnh sau để cài Zsh:
```sh
sudo apt install zsh
```
2. Đặt shell mặc định là Zsh bằng câu lệnh:
```sh
chsh -s $(which zsh)
```
3. Log out ra và login lại để kiểm tra Zsh là shell mặc định. Có thể chạy lệnh `echo $SHELL` để kiểm tra.

## 3. Cài Oh My Zsh
Chạy lệnh sau để cài Oh My Zsh:
```sh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

Mở file `~/.zshrc` để cấu hình:
```sh
nano ~/.zshrc
```

```
ZSH_THEME="agnoster"

plugins=(
  git
  git-flow
  colored-man-pages
  command-not-found
  debian
  extract
)
```

Cài font PowerLine để dùng với theme Agnoster: https://github.com/powerline/fonts

## 4. Plugins
### ZSH Syntax Highlighting
Repo: https://github.com/zsh-users/zsh-syntax-highlighting

Cài đặt:

1. Clone repo vào `~/.oh-my-zsh`:
```sh
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

2. Thêm plugin vào `~/.zshrc`:
```
plugins=( [plugins...] zsh-syntax-highlighting)
```

3. Mở lại Terminal.

### ZSH Autosuggestion
Repo: https://github.com/zsh-users/zsh-autosuggestions

Cài đặt:

1. Clone repo vào `~/.oh-my-zsh`:
```sh
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

2. Thêm plugin vào `~/.zshrc`:
```
plugins=( [plugins...] zsh-autosuggestions)
```

3. Mở lại Terminal.

### The Fuck
Repo: https://github.com/nvbn/thefuck

Cài đặt:

1. Cài đặt The Fuck bằng `pip`:
```sh
sudo apt update
sudo apt install python3-dev python3-pip python3-setuptools
sudo pip3 install thefuck
```

2. Thêm plugin vào `~/.zshrc`:
```
plugins=( [plugins...] thefuck)
```

3. Thêm dòng sau vào cuối `~/.zshrc`:
```sh
eval $(thefuck --alias)
```

4. Mở lại Terminal.

5. Cách sử dụng: Enter `fuck` hoặc bấm ESC 2 lần nếu command trước đó fail (VD: thiếu `sudo`).

### ZSH NVM
Repo: https://github.com/lukechilds/zsh-nvm

Cài đặt:
1. Gỡ bỏ `nvm` nếu đã cài: https://github.com/nvm-sh/nvm#uninstalling--removal

2. Clone repo vào `~/.oh-my-zsh`:
```sh
git clone https://github.com/lukechilds/zsh-nvm ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-nvm
```

3. Thêm plugin vào `~/.zshrc`:
```
plugins=( [plugins...] zsh-nvm)
```

4. Mở lại Terminal.

### ZSH Better NPM Completion
Repo: https://github.com/lukechilds/zsh-better-npm-completion

Cài đặt:

1. Clone repo vào `~/.oh-my-zsh`:
```sh
git clone https://github.com/lukechilds/zsh-better-npm-completion ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-better-npm-completion
```

2. Thêm plugin vào `~/.zshrc`:
```
plugins=( [plugins...] zsh-better-npm-completion)
```

Nếu trong danh sách plugin có `npm` thì xoá bỏ nó trước khi thêm vào.

3. Mở lại Terminal.
