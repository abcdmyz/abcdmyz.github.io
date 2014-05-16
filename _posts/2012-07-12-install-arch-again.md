---
layout: post
title: 再装Arch记
description : ""
category :
tags: []
---

# 再装Arch记

由于换了实验室，又要重新装系统。 本以为有了之前的各种经验，这次应该会快一点，谁知道，又花了大半天。 第一个很囧的问题是，装到一半，网络出问题了。就是各种连不上，ping什么都不通。因为实验室这里的网络是dhcp，一开始就担心会不会有各种问题。 因为对网络一窍不通啊。网络不通后就是各种配各种调，最后的最后发现，原来是网线松了，囧啊。 安装的流程跟之前的几乎差不多，但是装xorg的时候，有提示以下出错信息。 arch xf86-video-intel-sna and x86-video-intel-uxa are in conflict 一开始不知道什么东西，上网google，好象是显卡驱动。既然冲突，那就二选一装一个吧。选了uxa。但是装完以后，重启，结果，花屏。无奈，只能从u盘启动，然后把硬盘的东西mount上来，修改一下/etc/rc.conf，把最后的gdm去掉，就是不启动图形界面。卸载uxa，安装sna，同样花屏。 无果，请教高手。结果发现实验室的电脑的显卡不是intel的。用以下命令可确定显卡是nvidia的。 [shell]lspci | grep -i "nvidia"[/shell] 于是按照arch wiki上的方式安装显卡驱动，安装了一个开源的nvidia的驱动，nouveau。<https://wiki.archlinux.org/index.php/Nouveau> 其实具体装了什么，又忘了。只记得就这么乱糟糟地随便装了一堆东西。 [shell] yaourt -S xf86-video-nouveau yaourt -S --asdeps libgl modprobe nouveau [/shell] 记忆中有执行这么几条吧？如果之前有安装nvidia的驱动，要先卸载了。 装完这个驱动，总算能进入图形界面了。结果登录的时候，又弹出这么一个错误。 Could not update ICEauthority file /home/user/.ICEauthority 进入home下看，发现没有user目录。又继续google了。google出以下命令是可行的。 [shell]sudo chown user -R /home/user[/shell] 不过，chown是修改文件的所有者。猜测事因为home目录下没有user目录，所以会先创建，然后再把权限交给user？