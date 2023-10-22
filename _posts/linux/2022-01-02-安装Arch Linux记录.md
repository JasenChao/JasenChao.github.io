---
layout: post
title: 安装Arch Linux记录
tags: [linux, archlinux]
categories: 文章
---

* TOC
{:toc}

# 安装Arch Linux记录

1. 在U盘中安装镜像，U盘启动。

   无线网络安装时需要连接WiFi，有线网络安装则跳过这一步：

   ```shell
   rfkill unblock wifi
   iwctl
   station wlan0 connect <wifi_ID>
   station wlan0 show					//检查是否连接成功
   exit								//退出iwctl
   ```

   然后开始安装：

   ```shell
   archinstall
   ```

2. 按照引导选择每个选项，需要注意的是，网络配置要选择安装通用驱动，否则不能联网，如果是N卡，则在驱动安装步骤选择闭源驱动，开源驱动真的不行，尤其是独显直连，卡爆了。

3. 安装完成之后退出U盘，重启，此时在进入桌面之前，如果是N卡，可能无法进入，左上角选择wayland改为x11即可。

4. 进入后可能无法显示中文，所有的中文都是框框，需要安装中文字体：

   ```shell
   sudo pacman -S ttf-droid wqy-microhei wqy-zenhei noto-fonts-emoji ttf-font-awesome
   ```

5. 安装中文输入法：

   ```shell
   sudo pacman -S fcitx5 fcitx5-im fcitx5-chinese-addons fcitx5-rime
   ```

   修改环境变量`/etc/environment`，增加以下内容：

   ```
   GTK_IM_MODULE=fcitx
   QT_IM_MODULE=fcitx
   XMODIFIERS=@im=fcitx
   INPUT_METHOD=fcitx
   SDL_IM_MODULE=fcitx
   GLFW_IM_MODULE=ibus
   ```

   在设置，input中增加pinyin，重启电脑，ctrl + space即可切换中文输入。
   
6. 连接网络，在安装过程中网络环节选择安装NetworkManager，在命令行输入`nmtui`连接网络。

# 在Arch Linux中安装VMware

1. 下载VMware Player的安装包。是一个bundle文件，终端里面用sh执行它。

2. 安装好之后运行，可能会报错，需要安装：

   ```shell
   sudo pacman -S linux-headers
   ```

   之后他会自己编译安装两个组件，`vmmon`和`vmnet`，如果编译失败了，则需要我们自己安装：

   ```shell
   git clone https://github.com/mkubecek/vmware-host-modules.git
   cd vmware-host-modules/
   git checkout player-16.2.3		//根据vmware对应的版本选择
   make
   sudo make install
   ```

   再运行VMware，OK。
   
3. 如果报vmmon版本错误，说明编译时的分支选择不正确，重新编译安装后，重新加载vmmon模块：

   ```shell
   sudo modprobe -r vmmon
   sudo modprobe -a vmmon
   ```
   
4. 每次开机都要执行以下命令：

   ```shell
   sudo modprobe -a vmmon
   sudo /etc/init.d/vmware start
   ```

   

# 在archlinux中安装virtualbox

1. ```shell
   sudo pacman -S virtualbox		//选择dkm
   ```

2. ```shell
   sudo modprobe vboxdrv vboxnetadp vboxnetflt
   ```

   

# 在Arch Linux中创建应用快捷方式

1. 在`/usr/share/applications/`中新建文件`app.desktop`，app为快捷方式的名称

2. 文件内容如下：

   ```
   [Desktop Entry]
   Name=app																	//根据快捷方式名称修改
   Comment=xxx																	//应用的描述
   GenericName=Markdown Editor
   Exec=/home/why/program/typora/Typora										//应用所在目录
   Icon=/home/why/program/typora/resources/assets/icon/icon_256x256@2x.png		//应用图标位置
   Type=Application															//应用类型
   Categories=Office;WordProcessor;Development;								//应用归类
   MimeType=text/markdown;text/x-markdown;										//可打开的文件类型
   ```




# 在Arch Linux中安装v2rayA

1. GitHub上下载安装包：

   ```shell
   sudo pacman -U *.pkg.tar.zstd
   ```

2. 启动相关服务：

   ```shell
   systemctl enable --now v2raya.service
   ```

   如果升级安装，需要手动重启服务：

   ```shell
   systemctl restart v2raya.service
   ```




# Arch Linux美化neovim

1. 构建lua目录结构，在`~/.config/nvim`下：

   ```shell
   ├── ftplugin
   ├── init.lua
   ├── lint
   ├── lua
   │   ├── basic
   │   │   ├── config.lua
   │   │   ├── plugins.lua
   │   │   └── settings.lua
   │   ├── conf
   │   │   └── nvim-tree.lua
   │   ├── dap
   │   └── lsp
   ├── plugin
   │   └── packer_compiled.lua
   └── snippet
   ```

2. `init.lua`中调用其他lua文件：

   ```
   require("basic.settings")
   require("basic.config")
   require("basic.plugins")
   ```

3. `settings.lua`中写入设置：

   ```
   -- 设定各种文本的字符编码
   vim.o.encoding = "utf-8"
   -- 设定在无操作时，交换文件刷写到磁盘的等待毫秒数（默认为 4000）
   vim.o.updatetime = 100
   -- 设定等待按键时长的毫秒数
   vim.o.timeoutlen = 500
   -- 是否在屏幕最后一行显示命令
   vim.o.showcmd = true
   -- 是否允许缓冲区未保存时就切换
   vim.o.hidden = true
   -- 是否开启 xterm 兼容的终端 24 位色彩支持
   vim.o.termguicolors = true
   -- 是否高亮当前文本行
   vim.o.cursorline = true
   -- 是否开启语法高亮
   vim.o.syntax = "enable"
   -- 是否显示绝对行号
   vim.o.number = true
   -- 是否显示相对行号
   vim.o.relativenumber = true
   -- 设定光标上下两侧最少保留的屏幕行数
   vim.o.scrolloff = 10
   -- 是否支持鼠标操作
   vim.o.mouse = "a"
   -- 是否启用系统剪切板
   vim.o.clipboard = "unnamedplus"
   -- 是否开启备份文件
   vim.o.backup = false
   -- 是否开启交换文件
   vim.o.swapfile = false
   -- 是否特殊显示空格等字符
   vim.o.list = true
   -- 是否开启自动缩进
   vim.o.autoindent = true
   -- 设定自动缩进的策略为 plugin
   vim.o.filetype = "plugin"
   -- 是否开启高亮搜索
   vim.o.hlsearch = true
   -- 是否在插入括号时短暂跳转到另一半括号上
   vim.o.showmatch = true
   -- 是否开启命令行补全
   vim.o.wildmenu = true
   -- 是否在搜索时忽略大小写
   vim.o.ignorecase = true
   -- 是否开启在搜索时如果有大写字母，则关闭忽略大小写的选项
   vim.o.smartcase = true
   -- 是否开启单词拼写检查
   vim.o.spell = true
   -- 设定单词拼写检查的语言
   vim.o.spelllang = "en_us,cjk"
   -- 是否开启代码折叠
   vim.o.foldenable = true
   -- 指定代码折叠的策略是按照缩进进行的
   vim.o.foldmethod = "indent"
   -- 指定代码折叠的最高层级为 100
   vim.o.foldlevel = 100
   ```

4. `plugins.lua`中写入插件，写完后用`PackerSync`进行更新和安装：

   ```
   ---@diagnostic disable: undefined-global
   -- https://github.com/wbthomason/packer.nvim
   
   local packer = require("packer")
   
   packer.startup(
       {
           -- 所有插件的安装都书写在 function 中
           function()
               -- 包管理器
               use {
                   "wbthomason/packer.nvim"
               }
   
               -- 安装其它插件
   
               -- 中文文档
               use {
                   "yianwillis/vimcdoc",
               }
   
               -- nvim-tree
               use {
                   "kyazdani42/nvim-tree.lua",
                   requires = {
                           -- 依赖一个图标插件
                           "kyazdani42/nvim-web-devicons"
                   },
                   config = function()
                           -- 插件加载完成后自动运行 lua/conf/nvim-tree.lua 文件中的代码
                           require("conf.nvim-tree")
                   end
               }
   
           end,
           -- 使用浮动窗口
           config = {
               display = {
                   open_fn = require("packer.util").float
               }
           }
       }
   )
   
   -- 实时生效配置
   vim.cmd(
       [[
     augroup packer_user_config
       autocmd!
       autocmd BufWritePost plugins.lua source <afile> | PackerCompile
     augroup end
   ]]
   )
   ```

   插件中的`nvim-tree`会丢失字体依赖，在https://www.nerdfonts.com/#home中下载字体，将解压的ttf文件放入`/usr/share/fonts/TTF`中，用以下命令更新字体：

   ```shell
   fc-cache -vf
   ```

   

# 配置i3wm

1. i3的配置文件位于`~/.config/i3/config`，常用设置有：

   ```
   exec_always xxx			//总是启动xxx程序
   bindsym $mod+g exec google-chrome-stable		//设置快捷键启动chrome
   new_window 1pixel		//去掉程序边框
   gaps inner 5			//设置窗口间隙
   ```

2. 多个monitor如何配置：

   ```
   xrandr		//查看当前显示器
   xrandr --output HDMI-A-0 --mode 1920x1080 --same-as eDP --auto		//同步模式
   xrandr --output eDP --left-of HDMI-A-0 --auto		//扩展模式，选项有--left0of, --right-of, --above, --below
   ```

3. dmenu增加程序：

   ```
   echo $PATH		//查看路径，在路径中增加软链接
   ln -s [源程序] [链接程序]		//设置软链接，可以用dmenu快速打开
   ```

4. 更改键位映射：

   用`xmodmap -pke > ~/.xmodmap`命令把键盘配置写入文件，在文件中修改，再用`xmodmap ~/.xmodmap`命令更新配置。

5. 程序推荐：

   | 软件              | 作用                                                    |
   | ----------------- | ------------------------------------------------------- |
   | variety           | 更换壁纸                                                |
   | compton           | 背景透明                                                |
   | lxappearance      | 设置主题                                                |
   | xorg              | 更改硬件配置                                            |
   | dmenu             | 快速启动一个program launcher                            |
   | alacritty         | 终端程序，配置文件在`~/.config/alacritty/alacritty.yml` |
   | kdenlive          | 视频剪辑软件                                            |
   | gimp              | 修图                                                    |
   | vlc               | 视频播放软件                                            |
   | electronic-wechat | 微信                                                    |
   | transmission-qt   | 下载工具                                                |
   | qbittorrent       | bt下载工具                                              |
   | ranger            | 文件管理器                                              |
   | polybar           | i3顶部状态栏，配置很麻烦                                |

6. 调整屏幕亮度

   ```shell
   xrandr --output DP-4 --brightness 1		//DP-4取决于xrandr想调整的显示器，1可以是从0-1的小数，代表亮度
   ```

   

# yay

1. 安装 yay

   ```shell
   # 安装基本工具
   sudo pacman -S --needed base-devel git
   
   # 安装 yay
   git clone https://aur.archlinux.org/yay.git
   cd yay
   makepkg -si
   
   # 验证
   yay --version
   ```

2. 搜索软件包

   ```shell
   yay [searchterm]
   
   # 比如
   yay chrome
   
   # 在官方存储库和 AUR 上搜索，用 -Ss 选项
   yay -Ss google-chrome
   
   # 多关键字搜索，先搜 term1，再在结果中搜 term2
   yay -S term1 term2
   ```

3. 安装软件包

   ```shell
   yay -S [packagename]
   ```

4. 删除软件包

   ```shell
   yay -R packagename
   
   # 同时删除依赖
   yay -Rns google-chrome
   
   # 删除不必要的依赖
   yay -Yc
   ```

5. 升级软件包

   ```shell
   yay -Sua
   ```

6. 其他

   ```shell
   # 打印软件包统计信息
   yay -Ps
   
   # 查看帮助
   yay --help
   man yay
   ```

   