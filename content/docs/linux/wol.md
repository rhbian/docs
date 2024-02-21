---
title: "计算机的网络唤醒功能(WOL)"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 简介
网络唤醒，`Wake-on-LAN`，简写`WOL`。查阅一些资料，各种电脑的网络唤醒功能各异。我的笔记本没有此功能，有的电脑只支持睡眠状态的网络唤醒，有的则能支持完全关机的网络唤醒。很幸运这个迷你主机支持完全关机下的网络唤醒。下面是一些配置。

# 使用
1. BIOS  
在bios中找到有`wol`类似字样的设置并enable。
2. 软件查看 
查看现有设置。安装`ethtool`软件后，
```sh
ethtool eth0 | grep -i wake
```
`g`代表enable，`d`代表disable。

3. 永久修改
``` sh
# /etc/conf.d/net
ethtool_change_eth0="wol g"
```
`eth0`改为相应的网络接口。

4. 远程启动
    1. 查看本地网络接口和mac地址`ip addr`
    2. 另一台电脑运行 `wakeonlan <mac_address>`
