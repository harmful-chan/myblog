## 硬件资料

OpenWRT给出的：**https://openwrt.org/toh/hwdata/tp-link/tp-link_tl-wdr6500_v2**官网都没那么全

![image-20200523191456720](http://img.hfzs.store/myblog/img/image-20200523191456720.png)

## 一、刷系统

参考：**https://www.sunxidong.com/289.html**

需要的硬件软件：TP-LINK WDR6500 V2，Breed固件，OpenWRT WDR6500 专用固件

breed：**http://img.hfzs.store/file/openwrt/wdr6500v2.bin**

专用固件：**http://img.hfzs.store/file/openwrt/openwrt-18.06.2-ar71xx-generic-tl-wdr6500-v2-squashfs-factory.bin**

(自己家的服务器，心情不好能会关，有需要艾特一下我吧哈哈)

### 1. 烧breed

浏览器输入192.168.1.1(也可以是其他)打开tp-link配置页面，系统设置，更新固件，上传下载好的wdr6500v2.bin，点击更新，等待自动重启。

这时重启好之后还是之前的tp-link配置页面，需要按下reset键（有个圆形小孔）持续几秒钟，直到led不挺闪烁，松开reset，等待几秒钟，再输入网址192.169.1.1登录配置界面，就能看到breed的配置界面。

![1-1](http://img.hfzs.store/myblog/img/1-1.png)

### 2. 烧OpenWRT固件

先科普一下，factory与sysupgrade版本的区别。factory.bin = sysupgrade.bin + 剩余flash + 系统配置文件。sysupgrade是用来升级OpenWRT的，直接用breed烧进sysupgrade也能用，但是很需要技巧，本文最后说一下怎样ssh进入openwrt命令行重新烧进factory.bin

在breed 的固件更新界面，勾选固件上传xxx-factory.bin，自动重启，点击开始刷固件就行。过一会重新登录192.168.1.1就能看到OpenWRT的界面。首次登录用户名是root，没有密码，直接确定就行

![image-20200521232303731](http://img.hfzs.store/myblog/img/image-20200521232303731.png)

### 3. 汉化

web界面安装中文包。点击system下的software，filter 输入luci-i18n-base-zh-cn，点击查找，把找到的包装了会自动更新界面的。可能会遇到搜不出来的问题，可能是你wan口没插好，或者dns配置有误，本文最后会教如何解决。

![image-20200521233010076](http://img.hfzs.store/myblog/img/image-20200521233010076.png)

### 4. 设置LAN口和WAN口

基本上路由器wan口插上能外网的网线，lan口插上你的PC机，pc机网卡设置自动获取就能获得192.168.1.0/24的IP地址，没别的需求的话已经能用了。但往下我们需要配置pptp服务器，所以先把WAN口和LAN口设置一个静态IP。LAN口为192.168.1.1/24开DHCP，WAN口192.168.31.254/24关DHCP，因为我想外网访问所以wan要设置个IP。到此为止用web界面做的配置就配置完了，pc机也能正常上网了，下面开始命令行配置pptpd。

![image-20200523120845642](http://img.hfzs.store/myblog/img/image-20200523120845642.png)

## 二、配置pptp server

### 1. 连接OpenWRT的ssh

本文用的是xshell客户端，如果是win10系统的话直接在命令行敲`ssh root@192.168.1.1`因为win10自带了ssh客户端，openwrt默认的用户是root，IP默认是192.168.1.1，如果LAN设置了其他IP对着来改就是。然后敲yes保存公钥，然后输入密码（输入的密码是看不见的，密码默认为空，直接敲回车）。然后就连上了。

![image-20200523122826582](http://img.hfzs.store/myblog/img/image-20200523122826582.png)

### 2. 安装pptp

官方**https://oldwiki.archive.openwrt.org/doc/howto/vpn.server.pptpd#prerequisites**

```shell
$ opkg update    #更新一下包列表，如果失败或者报错，ping www.baidu.com 看看能不能连网，或者按照下面方法更新一下dns
$ opkg install pptpd kmod-mppe ppp    #最好分三条命令安装着三个包
```

下面是安装好的样子。

![image-20200523123432830](http://img.hfzs.store/myblog/img/image-20200523123432830.png)

然后就是配置pptpd了，官方有三个配置文件可以修改/etc/pptpd.conf，/etc/ppp/options.pptpd，/etc/ppp/chap-secrets但OpenWRT提供了一个更简单的配置方法直接配置/etc/config/pptpd，就可以满足日常需要

```shell
$ vi /etc/config/pptpd
#写入一下信息
config service 'pptpd'
        option 'enabled' '1'   #使能
        option 'localip' '192.168.1.1'    #连接VPN之后客户端看到的VPN服务器地址，应该不算是网关，但功能查不到
        option 'remoteip' '192.168.1.2-5'   #客户端分配地址范围
        option 'nat' '1'    #启用net装换，猜的
        option 'internet' '1'   #能和局域网通信，猜的

config 'login'    #能登陆的账号
        option 'username' 'aaa'
        option 'password' 'aaa'

config 'login'
        option 'username' 'bbb'
        option 'password' 'bbb'
```

下面是我的本机配置。

![image-20200523124820230](http://img.hfzs.store/myblog/img/image-20200523124820230.png)

然后就可以启动pptpd啦，然后看一下是否成功监听1723端口

```shell
$ /etc/init.d/pptpd enable    # 开启启动，这个是最直接的方法，和service pptpd enable，效果是一样的
$ /etc/init.d/pptpd start    # 启动
$ netstat -antp    # 查看端口1723
```

下面是我本机的例子。

![image-20200523125602980](http://img.hfzs.store/myblog/img/image-20200523125602980.png)

### 3. 连接vpn

我是用win10自带的vpn客户端，设置->网络和internet->vpn，设置IP（ip要是属于OpenWRT的，能ping通），用户名，密码，然后直接连接。然后没有意外是连不上的提示：端口已关闭之类的的错误。然后怎么办呢。

我是先开打cmd，telnet到OpenWRT的1723端口发现是通的，端口是没问题的，

![telnet](http://img.hfzs.store/myblog/img/telnet.gif)

百度了一下发现是18.x版本去除了个模块要手动安装，参考**https://blog.csdn.net/boliang319/article/details/49755701**，官方**https://oldwiki.archive.openwrt.org/doc/howto/vpn.nat.pptp**

```shell
$ opkg install kmod-nf-nathelper-extra
```

然后重启下网络和pptpd，连上了，皆大欢喜。然而很没有介绍，连上vpn之后发现不能连外网，需要配置iptables规则，参考**https://blog.csdn.net/d9394952/article/details/87868803**，官方**https://oldwiki.archive.openwrt.org/doc/howto/vpn.server.pptpd#prerequisites**

```shell
$ vi /etc/firewall.user    #用来配置用户规则的，主要规则在/etc/config/network
iptables -A forwarding_rule -i ppp+ -j ACCEPT 
iptables -A forwarding_rule -o ppp+ -j ACCEPT 
iptables -A output_rule -o ppp+ -j ACCEPT 
iptables -A input_wan_rule -p tcp --dport 1723 -j ACCEPT 
iptables -A input_wan_rule -p tcp --dport 47 -j ACCEPT 
iptables -A input_wan_rule -p gre -j ACCEPT 
iptables -A input_rule -i ppp+ -j ACCEPT     #上面简单来说就是配置允许ppp+和wan口的1723，47端口只能走tcp协议
```

然后练了vpn之后就能连网啦。但是还是不能访问局域网的其他主机，还要研究下。

#### 2020年6月2日17点20分更新

最近openwrt突然死了一次，可能是开太久了把，所以按照上面方法再做了一次。我最后直接是吧firewall給关掉了，但是用不敢把input，output规则设置成DROP，毕竟弄错了又要再来一遍，所以用我家的小米路由器做端口转发就算了，以下是iptables配置

首先先把防火墙关调，他是会自动清除iptables所有规则的
```shell
$ /etc/init.d/firewall stop && /etc/init.d/firewall disable
```
然后我们需要一条net转换规则用来给局域网内的机器连通外网，就是把局域网主机往外发的数据包改成右openwrt往外发。

```shell
$ iptables -t NAT -A POSTROUTING -s 192.168.1.0/24 -j MASQUERADE
```

为什么要用MASQUERADE呢，因为我发现直接`-j SNAT --to-source 192.168.31.254`(wan口地址)，

居于网主机是ping不通192.168.31.254的...尴尬。

再然后我们需要把服务器的9000端口映射带外网，所以首先我们要把小米路由的9000映射到192.168.31.254(openwrt wan口)，然后设置我们的iptables做端口转发，

```shell
$ iptables -t nat -A PREROUTING -p tcp --dport 9000 -j DNAT --to-destination 192.168.1.254:9000
$ iptables -t nat -A POSTROUTING -d 192.168.1.254 -p tcp --dport 9000 -j MASQUERADE
```

这样既能上网又能端口映射了。

#### 22点28分2020年6月9日更新

现在问题是内网机和客户端不能互访，客户机能ping内网机，内网机ping不同客户端。

经过几天的排查，发现了一些比较重要的问题，第一，连外网的nat转换没用指定网卡，所以把所有的192.168.1.0/24的包都SNAT成192.168.1.1所以正确的我们要这样修改

```shell
$ iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```

修改网之后全体人员都能连外网了，接下来就是互访问题，我们需要开启arp代理....打开设置文件` vim /etc/ppp/options.pptpd`添加以下一项

```shell
proxyarp    #客户端IP和内部网络在同一网段必须启用ARP代理
```

![image-20200609223430968](http://img.hfzs.store/myblog/img/image-20200609223430968.png)

然后重启pptpd`service pptpd restart`然后就好了。参考**https://www.oschina.net/question/139033_84155**

## 问题与解决

### opkg update 报错：* opkg_conf_load: Could not lock /var/lock/opkg.lock: Resource temporarily unavail

dns服务器解析不到。download.openwrt.org，改下dns就行

```shell
$ echo "nameserver 114.114.114.114">/tmp/resolv.conf
$ rm -f /var/lock/opkg.lock
$ opkg update
```

### OpenWRT命令行烧.bin固件文件

首先要把.bin文件拷到openwrt去，我用的是win10自带的scp命令，走22端口的。

```shell
$ scp xxx.bin root@192.168.1.1:/tmp/root    #我的就/tmp/root稍微大一点所以就考到这里了
```

然后进入OpenWRT进行烧写命令。

```shell
$ mtd -r write /tmp/root/xxxx.bin firmware    #这个命令是用来烧原厂固件的，也就是xxxfactory.bin
$ sysupgrade /tmp/root/xxx.bin    #设置是烧升级固件的，也就是xxxsysupgrade.bin
```

