---
layout: post
title: Tomato固件 + shadowsocks-libev - 实现对你路由的自动代理 
date: 2018-08-10 09:36:38

tags:
- GFW 
- VPN 

categories:
- Chinese
---


## 不用代理软件直连被墙网站

关于[GFW](https://zh.wikipedia.org/wiki/%E9%98%B2%E7%81%AB%E9%95%BF%E5%9F%8E)，大家可以wiki一下。它是你无法访问google的原因。我们的目的是在连上自家路由器wifi的条件下，就能够畅通访问所有被wall的网站。实现这个目的，大概需要这几样东西：

- 一个可以刷三方固件的路由器（比如，网件R7000）
- 一个在墙外的云主机（比如，Google Cloud Platform 的虚拟机实例）

<!-- more -->

## 云主机

云主机是关键-所有的被wall网站的访问都是走它的流量，使得能够访问这些网站。简单的说，就是在路由器识别出想要访问的网站为被wall网站的时候，会走路由器和云主机的线路，云主机代理请求，获得请求结果，再将结果给回路由器。因为云主机在墙外，这样所有连接该路由器的设备都能够获得翻墙的能力。

[Google Cloud Platform](https://cloud.google.com/free/?hl=zh-Cn)有一年免费（或者300美金）的赠款，就是一年期限先到或者300美金先用完，免费服务就停了。配置和申请云主机的过程需要能够连上google的服务，所以这变成了一个鸡生蛋、蛋生鸡的问题。如果要想办法薅这缕羊毛也需要你目前有一个能够翻墙的方案。

#### 免费云主机的申请流程

进入网站[GCP免费试用](https://cloud.google.com/free/?hl=zh-Cn)，点击免费使用。开始注册流程。

<div align='center'>
<img 
src="/images/ss_gcp_free_trail@2x.png" 
width="500" 
title = "注册页面"
alt = "注册页面"
align = center
/>
<br />
</div>

填写必要信息：如，地址、电话、信用卡等等。

<div align='center'>
<img 
src="/images/ss-gcp_free_trail2.jpg" 
width="650" 
title = "注册信息"
alt = "注册信息"
align = center
/>
<br />
</div>

成功注册之后就能够看到自己获赠的账户余额：

<div align='center'>
<img 
src="/images/ss-gcp_free_trail3.jpg" 
width="550" 
title = "注册信息"
alt = "注册信息"
align = center
/>
<br />
</div>

#### 创建虚拟机实例

可以不用纠结vm instance (虚拟机实例)到底是什么，只需要知道这是的安装shadowsocks服务端程序的服务器就好了。

去到GCP控制台，`计算引擎`栏下，选择`VM实例`。

<div align='center'>
<img 
src="/images/ss-gcp_free_trail4.jpg" 
width="450" 
title = "计算引擎"
alt = "计算引擎"
align = center
/>
<br />
</div>

<div align='center'>
<img 
src="/images/ss-gcp_free_trail5.png" 
width="480" 
title = "VM实例"
alt = "VM实例"
align = center
/>
<br />
</div>

点击创建实例，来创建安装shadowsocks服务端的程序。

<div align='center'>
<img 
src="/images/ss-gcp_vm_creation.png" 
width="450" 
title = "创建VM实例"
alt = "创建VM实例"
align = center
/>
<br />
</div>

在创建vm实例的时候，需要输入vm实例的名称，选择物理机房所在的位置。比较推荐的是地理距离较近的一些机房；比如，台湾、新加坡、日本等。

启动磁盘搭载的操作系统版本根据自己喜好选择熟悉的linux操作系统就好了。

在vm创建的展开项中，需要配置相应的安全项和网络项。

安全项主要是可以配置用户登录该实例的ssh key。这样就方便登录创建的vm实例，来安装shadowsocks程序。

<div align='center'>
<img 
src="/images/ss-gcp_vm_sercurity.png" 
width="450" 
title = "ssh登录配置"
alt = "ssh登录配置"
align = center
/>
<br />
</div>

当然，GCP本身也自带网页登录ssh的功能。ssh客户端登录更加方便，也不容易感受到网络延时的影响。

<div align='center'>
<img 
src="/images/ss-gcp_vm_static_ip.png" 
width="400" 
title = "静态ip配置"
alt = "静态ip配置"
align = center
/>
<br />
</div>

另外，我们需要在网络这个栏里面固定自己的ip地址。使用static ip而不是临时ip。

这些都配置好之后，就可以来安装shadowsocks服务端程序了。

#### 安装shadowsocks服务端程序

登录上刚刚创建的vm实例。可以在vm实例页面选择网页登录或者ssh客户端登录。

<div align='center'>
<img 
src="/images/ss-gcp_vm_login.png" 
width="550" 
title = "ssh登录"
alt = "ssh登录"
align = center
/>
<br />
</div>

```shell
sudo -i
apt-get install python-pip
pip install shadowsocks
```

如果在装shadowsocks的时候遇到下面的问题：

```
Traceback (most recent call last):
  File "/usr/bin/pip", line 11, in <module>
    sys.exit(main())
  File "/usr/lib/python2.7/dist-packages/pip/__init__.py", line 215, in main
    locale.setlocale(locale.LC_ALL, '')
  File "/usr/lib/python2.7/locale.py", line 581, in setlocale
    return _setlocale(category, locale)
locale.Error: unsupported locale setting
```

那需要修改一下locale配置，之后再装shadowsocks（`pip install shadowsocks`）:

```
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"
sudo dpkg-reconfigure locales
```

提示shadowsocks安装成功之后，就可以开始配置shadowsocks的服务端：

```
Collecting shadowsocks
  Downloading https://files.pythonhosted.org/packages/02/1e/e3a5135255d06813aca6631da31768d44f63692480af3a1621818008eb4a/shadowsocks-2.8.2.tar.gz
Building wheels for collected packages: shadowsocks
  Running setup.py bdist_wheel for shadowsocks ... done
  Stored in directory: /root/.cache/pip/wheels/5e/8d/b6/3e2243a7e116984b2c3597c122c29abcfeac77daa260079e88
Successfully built shadowsocks
Installing collected packages: shadowsocks
Successfully installed shadowsocks-2.8.2
```

我们需要去到`/etc`文件夹下创建一个shadowsocks的json配置文件，命名具有标识性就可以（比如，ss-conf.json）：

```
{
"server":"0.0.0.0",
"server_port":3389,
"local_address":"127.0.0.1",
"local_port":1080,
"password":"你想要配置的密码",
"timeout":600,
"method":"aes-256-cfb"
}
```

然后启动shadowsocks的服务：

```
ssserver -c /etc/ss-conf.json -d start
```

启动成功之后会有这样的提示：

```
INFO: loading config from /etc/ss-conf.json
2018-08-10 09:17:37 INFO     loading libcrypto from libcrypto.so.1.0.0
started
```

当服务端启动之后实际上你现在就能够翻墙了。如果是mac，可以使用ShadowsocksX-NG客户端软件来连接shadowsocks服务器。如果是其他平台，可以在[这里](https://shadowsocks.org/en/download/clients.html)找到相应的客户端软件。

<div align='center'>
<img 
src="/images/ss-gcp_client_config_entry.png" 
width="400" 
title = "ShadowsocksX-NG配置"
alt = "ShadowsocksX-NG配置"
align = center
/>
<br />
</div>

配置你的服务器参数，就可以翻墙了：

<div align='center'>
<img 
src="/images/ss-gcp_client_config.png" 
width="400" 
title = "ShadowsocksX-NG配置"
alt = "ShadowsocksX-NG配置"
align = center
/>
<br />
</div>

可以测试一下youtube、facebook的连接。

到这里，其实已经有一个自己的翻墙服务了。

下面半部分就是如何利用路由器无感知地翻墙。也就是只要是连上路由器wifi或者lan的设备都自然有翻墙的能力。

## 路由器配置

路由器配置这半部分相对于上面半部分有更多的内容，过程也相对复杂一些。

#### 可以刷三方固件(firmware)的路由器

路由器出厂都是自带原厂固件。路由器固件的概念，可以认为是路由器的操作系统；或者类比给iPhone刷机一样，就是更换iPhone硬件设备的固件。

现在比较流行的三方固件有[openWrt](https://openwrt.org/)、[merlin梅林](https://asuswrt.lostrealm.ca/)和[tomato](https://advancedtomato.com/)。我选择的是[Advanced Tomato](https://advancedtomato.com/)实际上就是在[原生Tomato](https://advancedtomato.com/)上添加了更友好的路由器管理界面。

因为我的路由设备是Netgear R7000，在openWrt的网站上有指出对这个型号的设备的wifi支持是有问题的，所以就没有敢刷openWrt。我个人会觉得openWrt是一个不错的选择。可以查看[openWrt的支持列表](https://openwrt.org/toh/views/toh_available_864)来购买相应的路由设备。

#### 刷固件

以我刷的Advanced Tomato为例，每一个固件的刷新大概经历三个过程：

- 使用原厂的固件配置好wifi，进入路由器设置页面。找到固件升级的选项。
- 将过渡固件通过设置页面上传到路由器，点击固件升级。开始刷过渡固件。刷完过渡固件一般需要清理一下路由的NVRAM。
- 最后刷最终固件。清理NVRAM。

具体过程可以参考[这个链接](https://www.youtube.com/watch?v=w10jLqRmdLM)。这个视频是针对原生tomato，但是AdvancedTomato只是界面风格不一样，菜单的选项都是大同小异。

#### 在路由器上安装shadowsocks客户端

你会发现不管是openWrt还是Tomato的固件，本质上只是一个mini Linux系统。而针对这些linux系统，可以同样适用ssh登录。也就是你可以ssh登录你的路由器。

比如，我的路由器的地址为192.168.2.1；因为天翼网关占了192.168.1.1。我可以在路由器的管理页面上配置ssh key:

<div align='center'>
<img 
src="/images/ss-router_ssh.png" 
width="550" 
title = "路由器ssh登录"
alt = "路由器ssh登录"
align = center
/>
<br />
</div>

配置好之后就可以`ssh root@192.168.2.1`：

```
➜ /Users/jiang >ssh root@192.168.2.1

Tomato v1.28.0000 -3.5-140 K26ARM USB AIO-64K
size: 36751 bytes (28785 left)
 ========================================================
 Welcome to the Netgear R7000 [TomatoUSB]
 Uptime:  16:45:30 up 8 days, 22:57
 Load average: 0.00, 0.01, 0.04
 Mem usage: 12.0% (used 29.88 of 249.64 MB)
 WAN : 192.168.1.3/24 @ B0:39:56:CD:4F:01
 LAN1 : 192.168.2.1/24 @ DHCP: 192.168.2.2 - 192.168.2.51
 WL0 : 2,4GHz @ Netgear-Wen @ channel: 6 @ B0:39:56:CD:4F:02
 WL1 : 5GHz @ Netgear-Wen5G @ channel: auto @ B0:39:56:CD:4F:03
 ========================================================

```

比较头疼的是，在Tomato上是没有包管理软件的。所以为了下载shadowsocks-libev（linux下的shadowsocks客户端），我给自己的tomato安装了entware来管理三方的pakage；而这里的shadowsocks-libev对于tomato而言就是所谓的三方软件包。

当然，你还有其他选择，比如optware。optware是类似entware的包管理软件。

在安装entware或者optware之前，需要路由器的flash memory来存储这些包。我选在将它存在了jffs上。关于将entware安装到jffs上，还是外部挂载的u盘上，可以看自己的路由器固件升级频繁不频繁。这些是jffs的一些相关内容：

```
http://tomatousb.org/doc:jffs

JFFS
Documentation » JFFS
What it is
In the tomato context, JFFS refers to a part of the internal flash memory in the router that can be used as additional storage space. It is the same internal flash where the Tomato firmware is installed.

The exact amount of available space in the internal flash depends on the flash overall size (varies on each router model) and on the installed build. Fully-featured builds in fact are fatter, use more flash and thus leave less space for JFFS. Lite builds, instead, have less features and are lighter, thus allowing for more space.

The data on the JFFS filesystem is not compressed. One entire flash block - usually 64kb - is reserved for overhead, plus an additional 1%-2% overhead per file.

When activated, JFFS is automatically mounted at startup on /jffs. This is done very early in the startup, before any other services (wan, lan, USB drives, cifs, etc.) are activated.

Problems with firmware upgrades
JFFS is very handy because it lets you install additional data within the router without having to rely on an external support. Alas, the main disadvantage of JFFS is that it prevents the router from being upgraded. In fact, the upgrade procedure must rewrite the whole flash, and the data in JFFS would be destroyed. TomatoUSB GUI helps you by not allowing an upgrade until JFFS is disabled. This means that you will have to backup the data you put in JFFS before doing an upgrade.

Amount of free space
JFFS is allocated within free blocks of flash memory. Each block is 64Kbyte, of which about 62Kbyte can be used for data. The following table shows the amount of free space available in JFFS for each flash size and each different build.

Std	Ext	Lite	No CIFS	VPN	No USB
JFFS space on 4Mb flash	300Kb	120Kb	600Kb	420Kb	0Kb	900Kb
So, if you have a router with 8Mb flash, you can check this table and add about 4Mbyte to those figures.

How to enable JFFS
Enable Administration » JFFS » Enable
Click on the Format / Erase button, and confirm the messagebox by clicking Yes.

Possible usages of JFFS
Store any kind of persistent data. If you write something to the standard filesystem (eg: /root), the changes will not persist after a router reboot. Data written under /jffs will instead be permanent.
Install optware, so to install useful packages.
Scripts to be executed when JFFS is mounted, via the same method as when a USB drive partition is mounted. All executable *.autorun files in /jffs (the top-level directory of the jffs filesystem) are executed.
```

明白了jffs和外接的存储的区别之后，我们开启jffs:

<div align='center'>
<img 
src="/images/ss-router_jffs_enable.png" 
width="480" 
title = "enable jffs"
alt = "enable jffs"
align = center
/>
<br />
</div>

同时登陆到路由器，在根目录的jffs下，创建一个opt目录；同时将jffs/opt目录挂载到根目录下的/opt目录下。这个步骤你可以不做，主要是为了在PATH中添加可执行文件路径的时候是/opt/bin:/opt/sbin/。

那为什么不直接安装到根目录下的/opt/文件夹下呢？因为它是一个只读目录。

```
cd /jffs
mkdir opt
cd ../
mount -o bind /jffs/opt /opt
```

这样，我们完成了/jffs/opt与根目录下的/opt的挂载。

接下来就是往/jffs/opt下安装entware，[关于entware](https://github.com/Entware/Entware/wiki)：

```
cd /opt
wget -O - http://qnapware.zyxmon.org/binaries-armv7/installer/entware_install_arm.sh | sh
```

<div align='center'>
<img 
src="/images/ss-router_entware_install.png" 
width="530" 
title = "entware 安装"
alt = "entware 安装 "
align = center
/>
<br />
</div>

成功安装entware之后，就可以使用entware来安装shadowsocks-libev：

```
opkg update
opkg install shadowsocks-libev
```

#### 配置shadowsocks客户端

当安装好了shadowsocks-libev之后，我们就可以配置shadowsocks客户端，让该客户端和你创建的google vm实例建立连接。

首先，我们在/jffs/opt/etc/下创建一个客户端配置json文件。该文件的格式和服务端配置的json文件非常类似。

```
vi /opt/etc/shadowsocks.json

root@unknown:/#
{
"server" : "google vm的静态ip",
"server_port" : "端口，默认3389",
"local_address" : "0.0.0.0",
"local_port" : "1080",
"password":"shadowsock当初配置的服务端密码",
"timeout":600,
"method":"shadowsock当初配置的服务端加密方式"
}

```

由于shadowsocks-libev的entware源只有`ss-redir   ss-rules   ss-tunnel`几个程序，没有常用的ss-local。我们将使用ss-redir来配置代理。

修改shadowsocks-libev默认的启动服务：

```
vi /opt/etc/init.d/S22shadowsocks
```

将`PROCS=ss-local`修改为`PROCS=ss-redir`。

我们这是可以尝试启动一下该服务，看能不能启动：

```
/opt/etc/init.d/S22shadowsocks start
```

#### 其他配置

刚才是ssh客户端登录了路由器之后，我们手动将/jffs/opt/挂载到了/opt/目录下。而这个挂载操作在路由器重启的时候就无效了；同样，我们启动的shadowsocks服务也是会这样。所以我们期望在路由器每次重启的时候,都执行挂载和shadowsocks服务的启动。我们需要将这些配置通过Tomato的启动页面配置到开机启动项中：

```
Administration >> Scripts >> init选项写入:

mount -o bind /jffs/opt /opt
/opt/etc/init.d/S22shadowsocks start
```

## 配置iptables和gfw列表

要让智能代理跑起来，有一个很重要的东西是：你期望对GFW屏蔽的网站（比如Youtube/Facebook/Google）走代理，而国内的网站不走代理。这样做有很多好处，最大的好处就是节省了你shadowsocks服务器的流量。

#### 配置gfw列表

对应屏蔽的网站来讲，我们需要能够在设备访问这些网站域名时，转换成正确的ip地址。比如你访问google.com的时候，访问的服务器地址可能是：216.58.221.78。我们dns服务器能够给你正确的可访问的ip地址。而国内的dns服务，会污染这些域名，返回给你一个无法到达的ip，这就达到了屏蔽该网站的目的。关于[dns防污染](https://zh.wikipedia.org/wiki/%E5%9F%9F%E5%90%8D%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%BC%93%E5%AD%98%E6%B1%A1%E6%9F%93)，这里有更多的信息。

我们这里采用的是确定的列表来应对dns防污染。*另外，还有通过udp转发来达到dns防污染的目的的方案。我没有采用，有兴趣可以研究一下。是优于这里使用确定的dns服务器的方案的。*

配置路由器：

```
Advanced >> DHCP/DNS >> DHCP / DNS Server (LAN) >> Dnsmasq
Custom configuration 框内写入 conf-dir=/jffs/dnsmasq.d
```

<div align='center'>
<img 
src="/images/ss-router_dnsmasq.png" 
width="500" 
title = "配置dnsmasq.d"
alt = "配置dnsmasq.d"
align = center
/>
<br />
</div>

ssh登录你的路由器：

```
root@unknown:/tmp/home/root# cd /jffs
root@unknown:/jffs# mkdir dnsmasq.d
root@unknown:/jffs# cd dnsmasq.d
root@unknown:/jffs/dnsmasq.d# vi dnsmasq.conf
```

编辑dnsmasq.conf文件，我将一份[更新好的列表放在了github上](https://github.com/Alex1989Wang/Dotfiles/tree/master/gfwlist2dnsmasq):

```
# dnsmasq rules generated by gfwlist
# Last Updated on 2018-08-04 23:26:15
# 
server=/030buy.com/208.67.222.222#443
ipset=/030buy.com/gfwlist
server=/0rz.tw/208.67.222.222#443
ipset=/0rz.tw/gfwlist
server=/1-apple.com.tw/208.67.222.222#443
ipset=/1-apple.com.tw/gfwlist
server=/10.tt/208.67.222.222#443
ipset=/10.tt/gfwlist
server=/1000giri.net/208.67.222.222#443
ipset=/1000giri.net/gfwlist
server=/100ke.org/208.67.222.222#443
ipset=/100ke.org/gfwlist
server=/10conditionsoflove.com/208.67.222.222#443
ipset=/10conditionsoflove.com/gfwlist
server=/10musume.com/208.67.222.222#443
ipset=/10musume.com/gfwlist
server=/123rf.com/208.67.222.222#443
ipset=/123rf.com/gfwlist
server=/12bet.com/208.67.222.222#443
ipset=/12bet.com/gfwlist
server=/12vpn.com/208.67.222.222#443
ipset=/12vpn.com/gfwlist
server=/12vpn.net/208.67.222.222#443
ipset=/12vpn.net/gfwlist
server=/138.com/208.67.222.222#443
ipset=/138.com/gfwlist
server=/141hongkong.com/208.67.222.222#443
ipset=/141hongkong.com/gfwlist
server=/141jj.com/208.67.222.222#443
ipset=/141jj.com/gfwlist
server=/141tube.com/208.67.222.222#443
ipset=/141tube.com/gfwlist
server=/1688.com.au/208.67.222.222#443
ipset=/1688.com.au/gfwlist

...

```

这里的208.67.222.222实际上是openDns的服务。

#### 配置iptables

在有了一个gfw列表之后，我们其实期望告诉路由器：如果你匹配到了gfw列表中的域名，请走openDns的dns(208.67.222.222)解析服务。

在shadowsocks的客户端配置中，我们指定了端口为`1080`；这里大概解释一下这些配置的意义：

```
{
"server" : "google vm的静态ip",
"server_port" : "服务端配置的端口 - 我的是3389",
"local_address" : "0.0.0.0",
"local_port" : "1080",
"password":"shadowsock当初配置的服务端密码",
"timeout":600,
"method":"shadowsock当初配置的服务端加密方式 - 我的是aes-256-cfb"
}
```
- "server"： 必须，服务器 IP 地址或域名。对应ss-redir命令行 -s 参数。
- "server_port"： 必须，shadowsocks 服务器所监听的端口。对应命令行 -p 参数。
- "local_address"： 必须，默认 "127.0.0.1"，由于我们需要在路由器上为网络中的各设备提供透明代理，此处应填入 "0.0.0.0"。对应命令行 -b 参数。
	- 要想使局域网内机器能够访问到部署在路由器上的 shadowsocks 服务，需要将该地址指定为路由器的IP地址（如 192.168.2.1 - 我的，具体值取决于所配置的路由器的 IP 地址）:
	- 要想使路由器自身的流量能够经过 shadowsocks 服务，需要将该地址指定为 127.0.0.1；
	- 若想使路由器自身和局域网内的机器都能够使用到 shadowsocks 服务，则需将该地址指定为 0.0.0.0。
- "local_port"： 必须，shadowsocks 客户端要监听的端口，该端口号还会在后面的的 iptables 规则配置中用到。对应命令行 -l 参数。
- "password"： 必须，shadowsocks 服务端所设置的密码。对应命令行 -k 参数。
- "method"： 可选，默认 "chacha20-ietf-poly1305″。应填入 shadowsocks 服务端所设置的加密方法。对应命令行 -m 参数。
- "mode"： 可选，默认 "tcp_only"。如服务端及客户端环境支持 UDP，可填入 "tcp_and_udp" 以同时启用 TCP 和 UDP。对应命令行默认（"tcp_only"）以及 -u ("tcp_and_udp") 和 -U ("udp_only") 参数。

理解了客户端配置的端口之后，再来配置iptables的规则：

```
#!/bin/sh
# Copyright (C) 2015 OpenWrt.org

insmod ip_set
insmod ip_set_bitmap_ip
insmod ip_set_bitmap_ipmac
insmod ip_set_bitmap_port
insmod ip_set_hash_ip
insmod ip_set_hash_ipport
insmod ip_set_hash_ipportip
insmod ip_set_hash_ipportnet
insmod ip_set_hash_net
insmod ip_set_hash_netport
insmod ip_set_list_set
insmod xt_set

ipset -N gfwlist iphash
iptables -t nat -A PREROUTING -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-port 1080
iptables -t nat -A OUTPUT -p tcp -m set --match-set gfwlist dst -j REDIRECT --to-port 1080
```

这个是从openWrt搬下来的配置。`insmod`用于内核加载`ipset`相关的模块。

关于iptables，我也没有搞明白。大概的意思是如果match到gfwlist的列表，我们就redirect到1080的port。而刚刚我们才知道，1080的port是被shadowsocks客户端监听的。那么这些流量将通过ss-redir客户端软件到达ss-server。

如果执行了上面脚本。到这里，你就应该能够智能翻墙了。

只是有两点缺点：

1. 我们用于设置iptables的以上脚本，只是当前我们配置的时候执行了一次。我们是期望有一个定时的检查任务，能够定期地检查上面脚本是否有添加好gfwlist。
2. 我们dnsmasq文件是手动更新下来，脚本转换；最后在上传至我们的路由器的。也就是没有办法定义自动更新。关于这个，可以查阅“配置自动获取 GFWList 并更新"相关内容。

问题2暂时不打算在这篇博文中记录下来。

对于问题1。Tomato的设置页面有可以配置定时任务的地方。

首先，将上面的scipt放在文件中。以`load_modules.sh`命名，存放在`/jffs/`文件夹下。

然后在相同目录下，再创建一个`check_ipset.sh`的script:

```
#!/bin/sh

# VPN check script

LOGTIME=$(date "+%Y-%m-%d %H:%M:%S")
CHAIN=`iptables -t nat --list | grep -i gfwlist`

if [ -z "$CHAIN" ]
then
        sh /jffs/load_modules
        echo 'chain has finished loading at ['$LOGTIME'].' >> /tmp/load_chain.log 2>&1

else
        echo 'no problem at ['$LOGTIME'].' >> /tmp/load_chain.log 2>&1
fi
```

该script主要就是在nat表下面看是否能够找到配置的gfwlist的set。如果找不到，就调用一下load_modules。

在ssh登录的路由器/jffs/目录下，将两个文件的属性修改为可执行文件：

```
chmod u+x load_modules check_ipset
```

同时将`check_ipset`配置在Tomato的管理页面下：

<div align='center'>
<img 
src="/images/ss-router_check_ipset.png" 
width="580" 
title = "ipset check"
alt = "ipset check"
align = center
/>
<br />
</div>

这样就算完成了。

