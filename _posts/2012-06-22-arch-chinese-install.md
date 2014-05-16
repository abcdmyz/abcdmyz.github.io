---
layout: post
title: Arch的中文显示与输入
description : ""
category :
tags: []
---

# Arch的中文显示与输入

刚装好的Arch，看不了中文，打开中文网站是各种乱码，更输入不了中文。所以，要进行各种安装和设置。 **1\. 修改locale.gen** 可以执行以下命令查看当前的编码 [shell]locale[/shell] 如果刚装完系统，显示的编码应该是C。 修改 /etc/locale.gen。将以下两句的注释去掉： zh_CN.UTF-8 UTF-8 en_US.UTF-8 UTF-8 然后执行以下命令： [shell]locale-gen[/shell] **2\. 修改启动项** 在 /etc/profile 中添加以下内容： LANG=en_US.utf8 export LC_CTYPE="zh_CN.utf8" 第一个设置是将编码设置为utf8, 第二个设置是可输入中文。 **3\. 安装字体** 装足够多的字体能避免各种乱码的出现。 这里选了以下几种字体： adobe cjk wqy-zenhei wqy-microhei wqy-bitmapfont **4\.  安装输入法** 这里选了好像是最受欢迎的小企鹅输入法。 arch的wiki上有详细的教程。其实安装过程也很简单。 <https://wiki.archlinux.org/index.php/Fcitx> 执行以下命令安装： [shell]yaourt -S fcitx[/shell] 为了让输入法更流畅，最好把cloudpinyin也装上。 然后根据wiki上的提示，把以下内容写到 ~/.xprofile 文件中。 #setup XIM environment export GTK_IM_MODULE=xim export QT_IM_MODULE=xim export XMODIFIERS="@im=fcitx" 最后，为了让电脑启动的时候也启动输入法。需要执行以下命令。 [shell] cp /usr/share/applications/fcitx.desktop ~/.config/autostart/ [/shell] 注意，这里的autostart可能一开始不存在，需要自己建文件夹。 如此执行完，中文显示和输入就应该没问题了。

## Comments

**[lpy](#15 "2012-07-22 05:03:08"):** 为什么都能装 adobe-cjk= =。就我不能...T^T

