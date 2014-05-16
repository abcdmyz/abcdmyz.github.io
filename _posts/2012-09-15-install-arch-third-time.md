---
layout: post
title: 第三次装Arch
description : ""
category :
tags: []
---

# 第三次装Arch

换了一台电脑，系统又要重装。 一开始装的是略为旧的版本,结果remote安装失败,因为新版本貌似和旧版本不兼容. 那只好用local安装,但是,安装完要更新的话,又出现各种麻烦. arch首页上写了要rm各种东西,但是,我装出来的系统竟然没有/fonts! 对于一个菜鸟而已,实在不知道怎么办,那只好...装新版本吧! 因为英语太水了,经常会不自觉跳过wiki上看不懂的部分,以至于对于大牛而言2分钟能装完的系统,我装了2天. 惭愧惭愧. 新的版本是2012.09.07更新的,真是"新鲜热辣"啊. 下载镜像,是dual,里面32位和64位都有了. 烧到U盘上. [shell]dd if=XXXX.iso of=/dev/sdX[/shell] 对于怎么确定U盘是sdX的问题,我只知道一个极度没有水平的方法,插之前看看有什么sdX,插之后看多了什么,那个就是U盘了. 烧完以后就开始装系统了. 结果痛苦就开始了.新版竟然没有那个给我这种SB用的安装向导,变成了命令行输入!! 又再次难为我这种菜鸟了. 只能打开arch wiki上的教程,一点点地试着来. 先是分区,设置文件系统,还有挂载,这三个都比较容易. 

  * **磁盘分区**
[shell]cfdisk[/shell] 可以进入一个菜单格式的分区界面. [shell]fdisk /dev/sda[/shell] 也是一个可以用来分区的命令. 
  * **设置文件格式**
mkfs -t TYPE_NAME /dev/sdaX 对于boot, [shell]mkfs -t ext3 /dev/sda1[/shell] 对于root和home, [shell]mkfs -t ext4 /dev/sda3[/shell]   [shell]mkfs -t ext4 /dev/sda5[/shell] 对于swap [shell]mkswap /dev/sda2 swapon /dev/sda2[/shell] 
  * **挂载**
root一定要挂载在/mnt上,并且是首先要挂在的,原因可查阅wiki. [shell] mount /dev/sda3 /mnt mkdir /mnt/home mount /dev/sda5 /mnt/home mkdir /mnt/boot mount /dev/sda1 /mnt/boot [/shell] 
  * **安装基础系统**
以上的步骤完成以后,就可以安装最简单的系统了. 不过是网络安装的,所以要先配置网络,配置的方法wiki上很详细. 实验室的电脑是dhcp,所以不用配置了. 为了加快速度,还可以修改一下/etc/pacman.d/mirrorlist,把china的源放在最前面. 安装系统命令: [shell]pacstrap /mnt base base-devel[/shell] 安装bootloader: [shell]pacstrap /mnt grub-bios[/shell] 

  * **生成fstab**
fstab包括了文件系统的信息,它记录了分区是如何被挂载在系统上的. [shell]genfstab -p /mnt >> /mnt/etc/fstab[/shell] 
  * **改变root**
[shell]arch-chroot /mnt[/shell] 执行完这条命令后,提示符会变成 sh-XXX#, 接下来可以配置语言,时间等等的东西,不过不配置也无所谓,以后装好了系统再弄也行. 不过以下这句很重要了,创建ramdisk [shell]mkinitcpio -p linux[/shell] 
  * **配置bootloader**
这个就是一开始我漏了以至于纠结很久的地方. [shell] grub-install --target=i386-pc --recheck /dev/sda cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo grub-mkconfig -o /boot/grub/grub.cfg[/shell] 最后设置密码 [shell]passwd[/shell] 退出chroot环境 [shell]exit[/shell] 然后,重启,即可.

## Comments

**[madper](#26 "2012-09-19 06:33:03"):** 那个极度没有水平的方法... 不就是我的方法吗.... 呃... 被吐嘈了.... 掩面...

