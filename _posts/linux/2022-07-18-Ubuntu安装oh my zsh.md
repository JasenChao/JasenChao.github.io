---
layout: post
title: Ubuntu安装oh my zsh
tags: [linux, Ubuntu]
categories: 文章
---

* TOC
{:toc}

安装zsh

```shell
sudo apt-get install zsh
```

将shell改为zsh

```shell
chsh -s /bin/zsh
```

接下来下载oh my zsh，网上查到的curl和wget都太慢了，还下不下来，可以使用github：https://github.com/robbyrussell/oh-my-zsh

但是github也太慢了，码云上有同步的镜像：https://gitee.com/mirrors/oh-my-zsh.git

```shell
sh -c "$(curl -fsSL https://gitee.com/mirrors/oh-my-zsh/raw/master/tools/install.sh)"
```

安装好了，重新打开终端

推荐两个插件

```shell
git clone https://github.com/zsh-users/zsh-autosuggestions.git $ZSH_CUSTOM/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM}/plugins/zsh-syntax-highlighting
vim ~/.zshrc
```

找到plugins=(git)，改为

```zsh
plugins=(
  git
  zsh-autosuggestions
  zsh-syntax-highlighting
)
```

保存后执行

```shell
source ~/.zshrc
```

也可以更换各种主题，例如

```zsh
ZSH_THEME="ys"
```