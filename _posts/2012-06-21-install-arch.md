---
layout: post
title: 装Arch记
description : ""
category :
tags: []
---

# 装Arch记

不知道为什么，无论装过多少次系统，每次装都感觉从来没装过似的，基本不记得怎么装。只能说，装得还不够多。 一直说要去用linux，又因为种种原因，没有坚持下去。毕设完毕，终于狠下心，该用linux了。 第一步，装linux。问了实验室的的同学，同学A建议我装gentoo，原因是自己编译自己安装，每一步细节都很清楚。同学B建议我装arch，原因是gentoo编译死你。对于一个一直把linux当做windows来用，连cd和ls都不会用的人，最终采纳了同学B的建议，装arch。 arch的安装其实挺傻瓜似的，就是一直选择一直选择。但不至于像ubuntu那种那么傻瓜。忍不住想吐槽一下arch的wiki，写得真的不够“科普”，尤其那种给初学者看的，我一开始真的好多没看懂。我跟同学B说，如果arch 的 wiki让我看完以后不用问你问题，那么就是真的写得超赞了。结果，同学B最近每天都被烦得要死，只能说他rp不好，碰到我这么一个“极品”。但又或许是我实在太菜鸟了，同时英语水平更菜，所以才看不懂wiki。  —————————————————————分区————————————————————— 整个装机过程，对从前某些稀里糊涂的概念终于有些理解了，其中一个是分区。 用习惯了windows，分区不过就是C盘、D盘、E盘啥的，顶多就是说个主分区、扩展分区、逻辑分区就没了。至少我是这样。对“挂载”这个概念是完全没有概念。直到这两天装系统，同学A给我解释了一遍又一遍，我终于懂一丁点了。 在linux下，至少有这么几个分区。 

/boot 它包含了操作系统的内核和在启动系统过程中所要用到的文件。

/ 放除了/boot以外的东西。例如/usr , /var , /tmp , /home等。如果这些目录不单独放在某个分区，那么它们就都放在 /下。 说起/这个分区，这个分区应该叫做根目录吧。一开始我一直和root用户混淆了，因为装完系统后，进入/这个目录，会有一个root子目录，我就一直很纳闷，为什么根目录里还有根目录。后来才懂，此根非彼根。/表示的是一个根目录，装了整个系统出了/boot以外的东西。root表示的是用户，一个具有超级权限的用户，而这个用户的内容就放在这个根目录下。所以/下会有root。（不知道有没有讲糊涂了）

swap 用做虚拟内存的分区。

因为打算装双系统，所以一共给了7个分区。 

/dev/sda1
Primary
100M
NTFS
Windows Reserve

/dev/sda2
Primary
80G
NTFS
C

/dev/sda3
Primary
100M
ext3
/boot

/dev/sda4
Extend

/dev/sda5
Logic
120G
NTFS
 D

/dev/sda6
Logic
2G
swap
 SWAP

/dev/sda7
Logic
100G
ext4
 /
可以使用 fdisk 对整个磁盘操作 [shell]fdisk /dev/sda[/shell] ——————————————————————挂载———————————————————— 分区完毕，接下来就是傻瓜安装法了。不知道为什么烧到u盘的arch出了点问题，本地安装不成功，只好选了core-remot。source选个edu.cn的会比较快，安装的包有base和base-level选，总之我选了一个没有make，没有gcc的包，囧。别忘了设置root密码。其实忘了也没啥。我第一次忘了，然后有人告诉我重装吧，结果装完以后，又有人告诉我，其实不用重装的， 挂载一下，然后换个root就可以了。 步骤大概如下： 插入那个装系统的u盘，在/mnt目录下建立一个arch的目录。 然后把硬盘上的root挂载在这个arch目录下。 [shell]mount /dev/sd7 /mnt/arch[/shell] 然后把硬盘上的proc也挂载到arch目录下 [shell]mount -t proc proc /mnt/arch/proc[/shell] 接着转换root [shell]chroot /mnt/arch[/shell] 这样，就通过u盘进入了硬盘的root了，最后就是修改密码 [shell]passwd[/shell] 这么一弄，对挂载又有点理解了。在linux下，所有的分区都是通过文件目录进行访问的，只要将这些分区挂载在文件目录下即可。下午还问了同学一个问题，如果我连windows都不想要了，那那个分区要怎么给linux？同学回答：挂载！可以把 /home挂载在那个C盘上，那么windows没了，linux就“扩容”了。就如此简单。还没有试过，先记下来。 —————————————————————yaourt————————————————————— 到这一步，一个最简单的系统就安装出来了。 接下来是安装yaourt，arch的包管理器。yaourt是pacman的封装，yaourt比起pacman有三个优势，第一个是私有源，第二个是可以同时开多个yaourt，而pacman只能有一个，第三个是yaourt搜索功能比pacman做得好。 安装yaourt的方法 首先是修改 /etc/pacman.conf。 添加以下内容： [arch linuxfr] Server = http://repo.archlinux.fr/i686 同时修改以下内容： SignalLevel = never 有多少个signallever就要设置多少个level。 然后执行以下命令更新源列表。 [shell]pacman -Syy[/shell] 最后执行以下命令安装yaourt [shell]pacman -S yaourt[/shell] 哦，还忘了一个。在安装前应该先执行这个。 [shell]pacman-key --init[/shell] 这个是设置pacman的密钥。 ————————————————————GNOME———————————————————— 到目前为止还是各种命令行，令我这个用习惯了windows的人有点不习惯，所以该装个图形界面了。选了GNOME。用yaourt安装，只要执行各种命令就好。 [shell] yaourt -S gnome yaourt -S gnome-extra yaourt -S dbus yaourt -S xorg [/shell] 接着修改 /etc/rc.conf，把图形化界面加入系统启动中。 DAEMONS = ( syslog-ng dbus network crond gdm ) 到此为止，一个看起来还靠谱的系统装出来了。其实还有很多要弄的。