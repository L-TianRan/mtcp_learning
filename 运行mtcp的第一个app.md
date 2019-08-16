 # 运行mtcp的第一个app

 ## 0.查看example
打开`mtcp/apps/example`文件夹，查看其中`epserver`的介绍。
``` epserver: a simple mtcp-epoll-based web server
    Single-Process, Multi-threaded Usage:
      ./epserver -p www_home -f epserver.conf [-N #cores] 
      ex) ./epserver -p /home/notav/www -f epserver.conf -N 8
```

## 1.创建www文件夹
我在`/home/username`下创建了一个`www`文件夹，新建了仨文件。
```
$ pwd
/home/username/www
$ ls
history  index.html  nonsense.txt
```

## 2.运行epserver
进入`mtcp/apps/example`文件夹，执行命令
```
sudo ./epserver -p /home/litianran/www/ -f epserver.conf -N 16
```
报错
```
...
EAL: Error - exiting with code: 1
  Cause: Cannot init mbuf pool, errno: 12
```
原因是我的`Hugepage`太小了，需要重新进行设置。
回到项目的根目录，重新执行
```
./setup_mtcp_dpdk_env.sh /home/username/mtcp/dpdk
```
首先需要`Remove hugepage mappings`移除原来的配置。选择对应的选项34
然后重新设置hugepage，选择`Setup hugepage mappings for NUMA systems`,对应选项22。这次输入一个大一些的值，
我输入了2048，之前值是64。我的服务器内存是62G，所以扛得住。

再次执行
```
sudo ./epserver -p /home/litianran/www/ -f epserver.conf -N 16
```
又报错
```
............
---------------------------------------------------------------------------------
Interfaces:
name: dpdk0, ifindex: 0, hwaddr: 9C:E3:74:16:FF:6F, ipaddr: 172.16.111.100, netmask: 255.255.255.0
Number of NIC queues: 16
---------------------------------------------------------------------------------
Loading routing configurations from : config/route.conf
fopen: No such file or directory
Skip loading static routing table
Routes:
Destination: 172.16.111.0/24, Mask: 255.255.255.0, Masked: 172.16.111.0, Route: ifdx-0
---------------------------------------------------------------------------------
Loading ARP table from : config/arp.conf
fopen: No such file or directory
Skip loading static ARP table
ARP Table:
(blank)            //这里是空的
---------------------------------------------------------------------------------
Initializing port 0... Ethdev port_id=0 nb_rx_queues=16 > 8    //错在这里了
EAL: Error - exiting with code: 1
  Cause: Cannot configure device: err=-22, port=0, cores: 16
```
于是我们执行
```
sudo ./epserver -p /home/litianran/www/ -f epserver.conf -N 8
                                                      //把16换成了8
```
成功运行！
但是还有arp表和route表没有配置。后续会讲到。
第一个mtcp下的app就这样成功运行了。
