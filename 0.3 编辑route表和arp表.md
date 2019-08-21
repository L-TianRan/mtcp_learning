# 编辑route表和arp表.md

由于网管周末没上班，我先把服务器上的route和arp配好。
上文已经提到，mtcp用的route表为`config/route.conf`，arp表为`config/arp.conf`。
咱们先进入`config/`文件夹，发现里面有俩示例文件`sample_route.conf`和`sample_arp.conf`。

## 创建配置文件
咱们直接把示例文件复制一下
```
$ cp sample_route.conf route.conf
$ cp sample_arp.conf arp.conf
```

## 修改`arp.conf`
打开`arp.conf`，里面已经有两条配置信息了。根据自己网卡的实际情况，来配置arp表。网卡的MAC地址可以用`ifconfig`命令查询。
我的配置如下。
```
ARP_ENTRY 2
192.168.111.100/32 9c:e3:74:16:ff:6f    //server 网卡
192.168.111.101/32 28:d2:44:e0:88:ec    //client 网卡
```
客户端可以招搬这个。

## 修改`route.conf`
其实现在服务器端只有一个dpdk网卡，但是无所谓。
我们把它修改成.
```
ROUTES 2
192.168.111.100/32 dpdk0
192.168.111.101/32 dpdk1
```
之后客户端再配置route表的时候就不能招搬这个了。

## 移动配置
```
$ pwd
/home/username/mtcp/          // 当前处在项目的根目录
$ cp -r config/ apps/example/config/            //移动到epserver应用所在目录
```
**每一个mtcp应用的二进制文件下都要有一个config目录，里面装着刚才修改的路由和arp表，这样应用才会读取到其中的配置**

## 配置效果
重新执行epserver
```
$ sudo ./epserver -p /home/litianran/www/ -f epserver.conf -N 8
```
```
---------------------------------------------------------------------------------
Loading routing configurations from : config/route.conf        //route配置生效
Routes:
Destination: 172.16.111.0/24, Mask: 255.255.255.0, Masked: 172.16.111.0, Route: ifdx-0
Destination: 192.168.111.100/32, Mask: 255.255.255.255, Masked: 192.168.111.100, Route: ifdx-0
Destination: 192.168.111.101/32, Mask: 255.255.255.255, Masked: 192.168.111.101, Route: ifdx--1
---------------------------------------------------------------------------------
Loading ARP table from : config/arp.conf       //arp配置生效
ARP Table:
IP addr: 192.168.111.100, dst_hwaddr: 9C:E3:74:16:FF:6F
IP addr: 192.168.111.101, dst_hwaddr: 28:D2:44:E0:88:EC
---------------------------------------------------------------------------------
```
