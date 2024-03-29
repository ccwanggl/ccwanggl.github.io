---
title: How to setting up a home NAS system with slow machine
date: 2016-11-16 22:28:48
tags:
  - NAS
---

合理利用家里闲置的旧机器，配置一个 NAS，主要用来同步我家小天使的照片，视频，更多功能随需开发即可。

<!--more  -->

八月初的时候，宝贝女儿出生，不想错过每一个值得记录的瞬间。现实的问题是手机拍照、录像很方便，iphone 手机拍照也够清晰，但是自己和老婆的手机的容量太小了（16GB），装了一些应用后，存储照片和视频的空间就不多了。

### 硬件准备

手头正好有个磐正的主板、一个 AMD 的 CPU、一个 1 TB 的硬盘。主板以前跑 windows 的时候存在 CPU 过热，导致自动关机的问题。但是跑 server 不需要图形界面，估计过热的问题不会很严重。就差内存和电源了，在某宝上 100 RMB 买了一条二手 4GB DDR3 内存条，39 RMB 买了一个电源，硬件备齐。

这个破机器的噪声有点大，机器的噪声主要是 CPU 风扇和电源风扇，如果两个都换成静音的话，噪音问题就基本解决了。如果考虑到用电问题的话，可用树莓派作为主机。


### 文件共享服务器的类型及选择

文件共享服务器有很多种，各有优缺点，使用场景不同，优势表现不同。

#### FTP Server

FTP 是很常见的服务器，很多自由软件的都有提供自己的 ftp 文件下载地址。但是 ftp 比较适合在不同的主机之间进行文件传输。使用 ftp 时，你无法直接修改主机上的数据，必须先下载，然后修改，接着在上传回主机。


#### NFS Server and CIFS

NFS 即常说的网络文件系统，好处是可以直接对远程主机上的数据进行修改，但是只能在 Unix Like 的机器之间互相分享数据。但是作为大多数的家庭用户，相信 windows 才是主流。CIFS 这货就是个网上邻居了，很显然缺点就是只能实现 windows 主机之间的数据分享。

#### Samba Server

Samba 就是为了解决『世界大统』而生的，它完美的解决了 Unix LIke 和 windows 之间的数据共享问题。

### 搭建 Samba server




>相信对于大多数的家庭用户而言，NAS 主要的作用是作为媒体存储中心，就我个人的应用场景中，NAS 的重要作用之一就是存储孩子和家人的照片及视频。主要的优点就是通过手机直接将照片和视频上传到 NAS 服务器上，另外，可以在手机上随时浏览整个媒体库，再有就是可以利用手机的投屏功能，在『大电视』上看照片、看视频。

但是，有一个问题是，往往孩子的照片老人的手机上存储的最多，而且浏览的次数较多。因为我的 NAS 没有做 RAID 架构，文件都在单盘上，如果 Samba 共享目录的权限设置不当，老人进行了误操作，就容易造成悲剧了。

<!-- more -->
### 配置前的准备

#### 文件的权限

文件的权限看似复杂，其实归结起来就是『文件的权限：控制用户对文件内容操作』，操作内容包括：读取、修改、删除等。

![](http://linux.vbird.org/linux_basic/0210filepermission//centos7_0210filepermission_2.gif)

##### 第一行代表文件的类型与权限

![](http://linux.vbird.org/linux_basic/0210filepermission//0210filepermission_3.gif)

- 常见的文件类型
  - 「d」：目录
  - 「-」：普通文件
  - 「l」：连接文件（相当于 windows 的快捷方式）
  - 「b」：块设备（常见的存储设备）
  - 「c」：键盘、鼠标、打印机等（一次性读取设备）

- 「rwx」参数的组合

  > - r：可读取
  > - w: 可写
  > - x: 可执行
  >
  > r、w、x 可分别用 4、2、1 表示，即可以用数字和表示文件权限。

  - 第一组表示 User 权限，即文件的拥有者
  - 第二组表示 Group 权限，即组权限
  - 第三组表示 Others 权限，即以上两种以外的用户对文件的操作权限

需要注意的是，所以的权限都是针对用户的。

- 文件的连接数
- 文件的拥有者
- 文件的所属群组
- 文件的大小，默认以 bytes 显示，可以加 `-h` 选项看看
- 文件的修改日期
- 文件名

##### 文件权限配置命令

相关命令有以下三个：

- chgrp <--- `change group`

改变文件的所属群组，是为了匹配文件的群组权限。修改前确保群组存在。比较常用的参数为 `-R` ， `R` 表示递归。
- chown <--- `change owner`
- chmod

使用 `chmod` 命令的形式有数字和符号两种，非常灵活，视个人喜好。


#### 目录的权限

权限对于目录意味着什么？

- `r` --- `read contents in directory`

可以使用 `ls` 命令列出目录中为文件清单。

- `w` --- `modify contens of directory`

  - 新建文件或目录
  - 删除文件或目录
  - 文件或目录改名
  - 文件或目录的移动
- `x` --- `access directory`

对目录的访问权限，表示可以进入目录。

#### 特殊权限

##### SUID

`SUID` 针对于二进制文件，使使用者暂时获得文件『拥有者』的权限。最常见的场景是 `passwd` 命令的执行。

##### SGID

当 SGID 作用于目录时，意义就非常重大了。当用户对某一目录有写和执行权限时，该用户就可以在该目录下建立文件，如果该目录用 SGID 修饰，则该用户在这个目录下建立的文件都是属于这个目录所属的组。

##### SBIT

当某一个目录拥有 SBIT 权限时，则任何一个能够在这个目录下建立文件的用户，该用户在这个目录下所建立的文件，只有该用户自己和 root 可以删除，其他用户均不可以。


那么，如何设置上面所说的三种权限呢？首先来介绍一点预备的知识，用数字来表示权限：

```
  4 表示 SUID
  2 表示 SGID
  1 表示 SBIT
```

### Samba 共享目录的权限配置

1. 建立文件夹 `mkdir dir1 dir2`
2. 建立用户组 `groupadd famliy`
3. `usermod` 将用户添加到组， `groups` 查看用户所属的组
4. 设置目录 `dir1` 属于 `famliy`， 且组权限为 `r-x`
5. 设置目录 `dir2` 权限为 `rwxrwsrwt`
6. 定期整理 `dir2` 中的文件合并到 `dir1`中
