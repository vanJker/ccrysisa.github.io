---
title: ""
subtitle:
date: 2024-01-24T18:59:56+08:00
# draft: true
# author:
#   name:
#   link:
#   email:
#   avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - Linux
  - deepin
  - Ubuntu
categories:
  - Toolkit
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRss: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: true
lightgallery: true
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

记录我所使用过的 Linux 发行版的心得体会。

<!--more-->

## 物理机

### 安装与配置

新手教学影片：

bilibili: [深度操作系统 deepin 下载安装 (附双系统安装及分区指引)](https://www.bilibili.com/video/BV1ZQ4y1C7n3/?vd_source=99b5a7ef7355e5c62fe79d489b7711ca)

### man & tldr

deepin 默认没有预装完整的 man 手册，需要手动安装:

```bash
$ sudo apt install manpages manpages-dev
```

> The tldr-pages project is a collection of community-maintained help pages for command-line tools, that aims to be a simpler, more approachable complement to traditional man pages.

安装 tldr:

```bash
$ sudo apt install tldr
```

### Vim & VS Code

```bash {title="~/.vimrc"}
syntax on
set relativenumber
set number
set cursorline
colorscheme default
set bg=dark
set tabstop=4
set expandtab
set shiftwidth=4
set ai
set hlsearch
set smartindent
map <F4> : set nu!<BAR>set nonu?<CR>
```

也可以参考 George Hotz 的 [极简 Vim 设定](https://github.com/geohot/configuration/blob/master/.vimrc)

这里列举一下本人 VS Code 配置的插件：

- **Vim**
- **VSCode Great Icons**
- **Tokyo Night**
- **Git History**
- **Even Better TOML**: toml 语法高亮
- **clangd**: C/C++ 语法服务
- **Native Debug**: 调试 C/C++
- **rust-analyzer**: Rust 语法服务
- **CodeLLDB**: 调试 Rust
- **Dependi**: 管理包依赖关系
- **Error Lens**: 更强大的错误提示

{{< admonition question >}}
rust-analyzer 插件可能会因为新版本要求 glibc 2.29 而导致启动失败，请参考这个 [issue](https://github.com/rust-lang/rust-analyzer/issues/11558) 来解决。
{{< /admonition >}}

### 效果展示

{{< image src="/images/tools/deepin-terminal-vim.png" caption="deepin Terminial Vim" >}}
{{< image src="/images/tools/deepin-dde-desktop.png" caption="deepin DDE Desktop" >}}

## FAQ

### 网络代理

使用项目 [clash-for-linux-backup](https://github.com/Elegybackup/clash-for-linux-backup) 来配置 Linux 发行版的网络代理。

```bash
$ git clone https://github.com/Elegybackup/clash-for-linux-backup.git clash-for-linux
```

> 在境内可以使用 [gitclone](http://gitclone.com) 镜像站来加快 clone 的速度。过程当中可能需要安装 curl 和 net-tools，根据提示进行安装对应的包即可。

安装并启动完成后，可以通过 `localhost:9090/ui` 来访问 Dashboard。

重启后可能会出现，输入密码无法进入图形界面重新返回登录界面，这一循环状况。这个是 Linux 发行版默认的 shell 是 dash，但位于 `/etc/` 路径下的 clash 服务脚本需要使用 bash 才能运行造成的，只需将默认的 shell 改为 bash 即可解决问题：

```bash
$ ls -l /bin/sh
lrwxrwxrwx 1 root root 9 xx月  xx xx:xx /bin/sh -> /bin/dash
$ sudo rm /bin/sh
$ sudo ln -s /bin/bash /bin/sh
```

如果你已经处于无限登录界面循环这一状况，可以通过 `Ctrl + Alt + <F2>` 进入 tty2 界面进行修改。