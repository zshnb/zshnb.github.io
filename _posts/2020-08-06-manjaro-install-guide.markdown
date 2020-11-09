---
layout: post
title: Manjaro安装指南
date: 2020-08-06 13:50
author: zsh
catalog: true
tags:
    - Linux
    - Manjaro
---

## 源设置

1. 首先更换源地址为国内地址，执行完会弹出一个框选择，一般选择前3个即可

   ```shell
   sudo pacman -Syy
   sudo pacman-mirrors -i -c China -m rank
   sudo pacman -Syyu	
   ```

2. 添加aur的源，aur就是各种第三方软件包的源，正是因为aur，arch linux才有如此多的软件可供下载

   ```shell
   编辑/etc/pacman.conf文件，加入下面的内容：
   [archlinuxcn]
   SigLevel = Optional TrustedOnly
   Server = https://mirrors.sjtug.sjtu.edu.cn/archlinux-cn/$arch
   ```

   然后执行`sudo pacman -Syy && sudo pacman -S archlinuxcn-keyring` 和 `sudo pacman -Su` 此时系统更新完毕，接下来需要安装必需软件



## 必需软件

### 输入法

1. 安装fcitx和谷歌拼音输入法：

   ```shell
   sudo pacman -S fcitx-googlepinyin
   sudo pacman -S fcitx-im
   sudo pacman -S fcitx-configtool
   ```

2. 设置环境变量，在`~/.xprofile`文件（如果文件不存在就新建一个）末尾加上：

   ```shell
   export GTK_IM_MODULE=fcitx
   export QT_IM_MODULE=fcitx
   export XMODIFIERS="@im=fcitx"
   ```
   
3. 重启电脑后在命令行中执行fcitx

### Shell

1. 安装oh my zsh `sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"`

2. 安装powerline shell

   ```
   pip install powerline-shell // 没有pip先安装pip
   ```

   然后在.zshrc文件中添加以下内容

   ```
   function powerline_precmd() {
       PS1="$(powerline-shell --shell zsh $?)"
   }
   
   function install_powerline_precmd() {
     for s in "${precmd_functions[@]}"; do
       if [ "$s" = "powerline_precmd" ]; then
         return
       fi
     done
     precmd_functions+=(powerline_precmd)
   }
   
   if [ "$TERM" != "linux" ]; then
       install_powerline_precmd
   fi
   ```

   执行 `source .zshrc`

   安装自动补全和高亮插件

   - ```shell
     git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
     git clone https://github.com/zsh-users/zsh-syntax-highlighting ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
     ```

   - 编辑.zshrc，添加 `plugins=(zsh-autosuggestions zsh-syntax-highlighting)`

## 常用软件

- Firefox（自带）
- Java8
- Docker，docker-compose
- Electron-ssr
- Typora
- Vlc
- Telegram
- Intellij idea
- Data grip
- Postman
- Redis desktop manager
- Deepin-wine-wechat（安装前需安装fateroot, binutils, patch包）
- Chrome
- Albert
- Onedriver
- Wps
