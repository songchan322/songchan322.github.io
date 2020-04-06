---
title: Proxmox VE 多合一家庭服务器
date: 2020-04-01 13:21:57
tags: ["proxmoxVE","pfsense","clash","黑群晖"] 
banner_img: https://s1.ax1x.com/2020/04/06/G6PFWq.png
index_img: https://s1.ax1x.com/2020/04/06/G6PFWq.png
---
以前折腾过一段时间的PVE，但由于经验不足，碰到了一些当时觉得不能解决的问题，比如pfsense网速慢、黑群晖网速跑不满千兆等，后来就玩基于矿难蜗牛星际的黑群晖、基于斐讯N1的旁路由等等。

由于一次偶然的意外（主机电源炸了-_-!!），面临简单换一块电源还是折腾，我毅然决然地选择了折腾，看到某宝上hp800g3sff准系统3xx的价格貌似有点香，于是果断买买买。准系统弄回来后成色有点喜出望外，风扇上一点灰都没有，本打算继续黑苹果，结果死活引导不进系统。

于是梳理了下需求，目前本人在学习Web前端开发，而Windows下的CMD.exe实在是丑出天际以及没有趁手的命令行工具，好在微软已经意识到这个问题，借助WSL、VSCode Remote等工具可以很好地满足我这方面的需求。此外，看别人玩多合一家庭服务器手痒痒，于是把这台主机设定为软路由、透明代理网关、网络存储、远程开发机于一体的家庭服务器。好，下面开搞。

### 0.1 硬件
|部件|型号|备注|
|---|---|---|
|CPU|i5-7500|
|主板电源机箱|hp 800 g3 sff 准系统|
|内存|镁光8G x 2|
|网卡|intel i219 板载|PVE管理口|
|网卡|intel i350t4 寨卡|pfsense WAN/dsm LAN|
|存储1|ADATA 128G|PVE系统盘|
|存储2|intel 600p 256G|虚拟机专用存储盘|
|存储3|杂牌固态120G|群晖下载盘|
|存储4|西数紫盘4T|群晖仓储盘|

### 0.2 网络拓扑

由于需要使用IPTV但又懒，暂时由光猫负责拨号，ProxmoxVE承担多项功能，肩负软路由、透明代理网关、网络存储、远程开发机等任务，8口POE交换机除了交换机的本职工作还要给AP供电，图上少画了AC控制器，实际是有的。主力机是一台荣耀笔记本，搭配24寸2k外接显示器基本满足目前的使用需求。

![网络拓扑](https://s1.ax1x.com/2020/04/04/G0N3Jf.png)

### 1 安装Proxmox VE宿主
#### 1.1 下载镜像并制作U盘启动盘
首先从Proxmox VE的[官网](https://www.proxmox.com/en/downloads)下载6.1版本的ISO安装包，使用你喜欢的工具将其写入U盘，这里我使用的是Etcher。

![Etcher](https://s1.ax1x.com/2020/04/06/G6PPFs.png)

#### 1.2 图形化安装
将U盘插到主机上，开机选择从U盘启动，进入图形化安装界面，根据提示安装。

![安装界面](https://s1.ax1x.com/2020/04/06/G6PFWq.png)

稍微解释下这个管理口配置，对于有软路由需求的情况而言，建议将管理口设置成LAN网段，方便后期管理。

![管理口设置](https://s1.ax1x.com/2020/04/06/G6PElV.png)

安装好后，找一台有浏览器的设备，使用网线连上PVE的管理口，将设备ip设置成管理口同网段，通过浏览器访问`https://192.168.x.x:8006`便可进入PVE的web管理界面。

![登录](https://s1.ax1x.com/2020/04/06/G6PiYn.png)

#### 1.3 删除企业源
需删除企业源，否则apt update会失败。
```bash
rm -rf /etc/apt/sources.list.d/pve-enterprise.list
```

#### 1.4 去除订阅提醒
每次登陆web管理页面时会提醒订阅，可以通过修改`/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js`文件去除提醒。

在shell中打开文件
```bash
nano /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
```
找到
```js
if（data.status !== 'Active'）
```
替换成
```js
if（false）
```
重启服务以生效
```bash
systemctl restart pveproxy.service
```

#### 1.5 更换国内源
使用国内的debian源以提高下载速度。

在shell中打开文件
```bash
nano /etc/apt/sources.list
```
修改为
```bash
#deb http://ftp.debian.org/debian buster main contrib
#deb http://ftp.debian.org/debian buster-updates main contrib
# security updates
#deb http://security.debian.org buster/updates main contrib

# debian aliyun source
deb https://mirrors.aliyun.com/debian buster main contrib non-free
deb https://mirrors.aliyun.com/debian buster-updates main contrib non-free
deb https://mirrors.aliyun.com/debian-security buster/updates main contrib non-free

# proxmox source
# deb http://mirrors.ustc.edu.cn/proxmox/debian/pve buster pve-no-subscription
deb http://download.proxmox.com/debian/pve buster pve-no-subscription
```
更新&升级
```bash
apt update & apt upgrade
```

### 2 安装pfsense
pfSense是基于FreeBSD的、开源中最为可靠（World's Most Trusted Open Source Firewall）的、可与商业级防火墙一战（It has successfully replaced every big name commercial firewall you can imagine in numerous installations around the world）的防火墙。

#### 2.1 下载ISO安装盘
从[官网](https://www.pfsense.org/download/)下载IOS安装盘，并将其上传至PVE的local存储区。

![pfsense下载页](https://s1.ax1x.com/2020/04/06/G6Pnw4.png)

#### 2.2 创建虚拟机
按照下图配置创建虚拟机

![pfsense虚拟机](https://s1.ax1x.com/2020/04/06/G6PmmF.png)

创建完毕后不要立即启动，在虚拟机硬件菜单中增加一张网卡用作WAN口，记住网卡名称，后面需要用到。

网卡增加完毕后，启动虚拟机，一切正常的话可以在控制台中看到如下界面。

![pfsense启动页](https://s1.ax1x.com/2020/04/06/G6PVyT.png)

安装程序会自动运行，随后提醒是否重启。

#### 2.3 配置网络

虚拟机重启后会进入网络端口分配界面。

![pfsense网络配置](https://s1.ax1x.com/2020/04/06/G6Plf1.png)

根据实际情况指定LAN口和WAN口，其中LAN需配置静态IP地址。

#### 2.4 切换中文界面
使用浏览器访问刚才设置的LAN地址，初始用户名/秘密是admin/pfsense。
进入`System / General Setup`页面，将`Language`改为`Chinese(Simplified, China)`保存生效。

#### 2.5 禁用硬件校验和卸载（！！非常重要）

进入`系统 / 高级选项 / 网络设置`，勾选`禁用硬件校验和卸载`,否则网速会异常地慢。

![禁用硬件校验和卸载](https://s1.ax1x.com/2020/04/06/G6PMk9.png)

*关于pfsense的其他问题可以参考[官方文档](https://docs.netgate.com/pfsense/en/latest/virtualization/virtualizing-pfsense-with-proxmox.html)或Google。*

### 3 安装clash旁路由

旁路由的本质是一台网关服务器，在其上运行相应的代理软件，其他设备将其网关配置为该旁路由IP即可实现透明代理。下面基于Debian系统上运行Clash代理软件实现旁路由的功能。

#### 3.1 创建Debian虚拟机
安装过程请自行Google，安装完毕后设置成静态IP地址。
```bash
sudo nano /etc/network/interfaces
```
配置参考如下
```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens18
iface ens18 inet static
  address 192.168.4.2
  netmask 255.255.255.0
  gateway 192.168.4.1
  dns-nameservers 192.168.4.1
```

#### 3.2 下载clash
```bash
# 下载最新版 clash，注意根据自己的系统下载对应的版本，我的是 64 位的，所以下载的是 linux-amd64 这个版本
wget https://github.com/Dreamacro/clash/releases/download/v0.19.0/clash-linux-amd64-v0.19.0.gz
# 解压并且把二进制文件放到 /usr/bin ，并且加上可执行权限
gzip -d clash-linux-amd64-v0.19.0.gz
sudo mv clash-linux-amd64-v0.19.0 /usr/bin/clash
sudo chmod +x /usr/bin/clash
# 为 clash 添加绑定低位端口的权限，这样运行 clash 的时候无需 root 权限
sudo setcap cap_net_bind_service=+ep /usr/bin/clash
```

#### 3.3 创建配置文件
```bash
# 创建文件夹
mkdir -p ~/.config/clash
cd ~/.config/clash
# 创建配置文件
touch config.yaml
nano config.yaml
```
配置文件的内容我没有仔细去研究，因为我的机场服务商提供了clash的托管服务，浏览器访问该托管地址就是详细的clash配置。

配置好的config.yaml应该长这样。

![config.yaml](https://s1.ax1x.com/2020/04/06/G6PuTJ.png)

正确配置后，在`~/.config/clash`目录下输入`clash`命令，应该就可以正常运行了。

#### 3.4 设置转发和路由
编辑`/etc/sysctl.conf`文件开启转发功能，找到 `#net.ipv4.ip_forward=1`，去掉前面的#号

```bash
sudo nano /etc/sysctl.conf
## ------分割线------- ##
net.ipv4.ip_forward=1
```

依次执行下列iptables命令(可能需要使用`sudo`)

```bash
iptables -t nat -N clash
iptables -t nat -N clash_dns

iptables -t nat -A PREROUTING -p tcp --dport 53 -d 198.19.0.0/24 -j clash_dns
iptables -t nat -A PREROUTING -p udp --dport 53 -d 198.19.0.0/24 -j clash_dns
iptables -t nat -A PREROUTING -p tcp -j clash

# 这里需要注意的是，下面两行最后的 192.168.1.80 是当前旁路由的 IP 地址，请根据你自己的实际情况修改
# 如果你自己的旁路由 IP 跟下面的 IP 地址不对的话会造成无法翻墙
iptables -t nat -A clash_dns -p udp --dport 53 -d 198.19.0.0/24 -j DNAT --to-destination 192.168.1.80:53
iptables -t nat -A clash_dns -p tcp --dport 53 -d 198.19.0.0/24 -j DNAT --to-destination 192.168.1.80:53

iptables -t nat -A clash -d 0.0.0.0/8 -j RETURN
iptables -t nat -A clash -d 10.0.0.0/8 -j RETURN
iptables -t nat -A clash -d 127.0.0.0/8 -j RETURN
iptables -t nat -A clash -d 169.254.0.0/16 -j RETURN
iptables -t nat -A clash -d 172.16.0.0/12 -j RETURN
iptables -t nat -A clash -d 192.168.0.0/16 -j RETURN
iptables -t nat -A clash -d 224.0.0.0/4 -j RETURN
iptables -t nat -A clash -d 240.0.0.0/4 -j RETURN

iptables -t nat -A clash -p tcp -j REDIRECT --to-ports 7892
```

执行完上面的指令后需要安装iptables-persistent以保存iptables，根据提示保存即可。
```bash
sudo apt install iptables-persistent
```

#### 3.5 配置开机自启动
这里使用`systemctl`来管理clash服务。
首先创建clash服务文件

```bash
sudo touch /etc/systemd/system/clash.service
sudo nano /etc/systemd/system/clash.service
```
配置如下（修改`你的用户名`）

```bash
[Unit]
Description=clash daemon

[Service]
Type=simple
User=你的用户名
ExecStart=/usr/bin/clash -d /home/你的用户名/.config/clash/
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

保存，启动服务以生效
```bash
# 启动 clash
sudo systemctl start clash.service
# 启动开机自启
sudo systemctl enable clash.service
```

#### 3.6 使用
在对应的设备上，将网关地址设置成该旁路由的IP地址即可。

*如果遇到问题，可以在~/.config/clash目录下输入clash命令直接运行clash查错，我就遇到了`Country.mmdb无效`的问题*

### 4 安装群晖
To be contine