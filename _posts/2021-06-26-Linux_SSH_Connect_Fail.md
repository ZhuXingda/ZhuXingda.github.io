---
layout:       post
title: SSH一段时间不连，过一会再连就连接超时
subtitle:     ""
date:         2021-06-26
author:       "ZhuXingda"
header-mask:  0.3
catalog:      true
multilingual: false
comments: true
tags:
    - Linux
---

装了个Debian到没有显卡的主机上做内网服务器，每天早上来用SSH连这个服务器都会报连接超时失败，在服务器上用键盘curl请求一次百度过后就又能连接。查找解决办法，一开始以为是系统休眠了，按照 https://blog.csdn.net/longe329/article/details/109149663 的方法关闭了systemd的休眠功能。下午发现没奏效，每次在服务器上curl百度的时候第一次请求都会比较慢，大概一两秒才有返回，后面看其他解决办法说是因为无线网卡休眠了，这个原因相更像。解决办法是grub的配置文件里关闭ASPM https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/power_management_guide/aspm 

添加quiet pcie_aspm=off然后update-grub

```bash
# If you change this file, run 'update-grub' afterwards to update
# /boot/grub/grub.cfg.
# For full documentation of the options in this file, see:
#   info -f grub -n 'Simple configuration'

GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet pcie_aspm=off"
GRUB_CMDLINE_LINUX=""
```

