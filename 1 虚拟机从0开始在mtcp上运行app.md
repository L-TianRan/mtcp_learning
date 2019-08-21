# 虚拟机从0开始在mtcp上运行app
因为所里网络的原因，我最终还是打算开两台虚拟机跑mtcp了。所以得从0开始搭建mtcp环境。在mtcp上进行客户端服务器通信。

## 0. 查看环境
```
$ uname -a
Linux ubuntu 4.15.0-20-generic #21-Ubuntu SMP Tue Apr 24 06:16:15 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
//这是我虚拟机的环境
```
## 1. clone代码
```
$ pwd
/home/username/Desktop
$ git clone https://github.com/mtcp-stack/mtcp
```

## 2. 安装子模块
我们先不装库，我还搞不清库的名字叫啥。
```
$ git submodule init
$ git submodule update
```
## 3. 查看pc信息
要看看网卡是不是支持dpdk，[查看支持列表](https://core.dpdk.org/supported/)
```
$ lspci | grep "net"
//我的是intel 82545,正好支持。
```
再看看是不是采用了NUMA架构，看看有没有这些文件信息。
[NUMA架构下的CPU拓扑](https://blog.csdn.net/weijitao/article/details/52884422)。
```
$ cat /sys/devices/system/node/node0/numastat 
numa_hit 1682354
numa_miss 0
numa_foreign 0
interleave_hit 40568
local_node 1682354
other_node 0
```
所以我的PC是NUMA架构。

## 4. 装库
装库这个避免不了，不然编译通不过。依次执行以下命令。
```
$ sudo apt-get install libdpdk-dev libnuma-dev libgmp-dev libelf-dev
                                              //后面这个elf库是后面又报错得到的，现在先在这里装上
```
至于`libpthread`、`librt`，我Ubuntu上本来就有。
```
$ ls /usr/lib/x86_64-linux-gnu/ | grep "pthread"
```
```
// 结果
libpthread.a
libpthread_nonshared.a
libpthread.so
```
如果没有`pthread`库，我找到了一个方法，[装pthread](https://stackoverflow.com/questions/35932258/how-to-install-libpthread-a-in-ubuntu-14-04)。
其他库暂时没找到安装方法。
```
$ sudo apt-get install libpthread-stubs0-dev
```
然后还有一个内核头，执行`sudo apt-get install linux-headers-$(uname -r)`

## 5. 编译并配置
现在看看能不能编译通过，进入项目根目录。
```
$ ./setup_mtcp_dpdk_env.sh /home/li/Desktop/mtcp/dpdk
                       //后面这个路径一定要用绝对路径
```
接着往下执行选项进行配置
```
 Press [15] to compile x86_64-native-linuxapp-gcc version
 Press [18] to install igb_uio driver for Intel NICs  
 Press [22] to setup 2048 2MB hugepages             //这次在我虚拟机上我设置的大小是512，因为我虚拟机内存才2G
 Press [24] to register the Ethernet ports        //所绑定的网卡必须是未使用的
 Press [35] to quit the tool          //选yes
```
在配置过程中又爆了几个常见错误，缺少`make`、`gcc`和`python`。直接`sudo apt-get install make gcc python`.
 * 绑定网卡时提示所绑定的网卡状态为`active`，绑定失败。
需要把网卡给停掉。当然如果你有好几块网卡的话直接绑定未使用的网卡就行。
```
$ sudo ifconfig ens33 down
    //ens33是我的网卡名，需要改成你自己的
```
我虚拟机只配置了一块网卡，停了之后就没法联网了，需要联网的时候必须把所有配置都取消，还原到初始状态，后面会讲怎么还原。

## 6. 启用并配置网卡
根据自己的实际情况配置IP，以下是我的配置。
```
# 服务器（就是执行epserver的那台）
$ sudo ifconfig dpdk0 192.168.198.131 netmask 255.255.255.0 up
$ export RTE_SDK=`echo $PWD`/dpdk         //这一步在项目根目录下执行
$ export RTE_TARGET=x86_64-native-linuxapp-gcc
```
```
# 客户端（就是执行epwget那个）
$ sudo ifconfig dpdk0 192.168.198.130 netmask 255.255.255.0 up
$ export RTE_SDK=`echo $PWD`/dpdk         //这一步在项目根目录下执行
$ export RTE_TARGET=x86_64-native-linuxapp-gcc
```

## 7. 装mtcp库
```
./configure --with-dpdk-lib=$RTE_SDK/$RTE_TARGET         //同样在项目根目录下执行
```
提示check成功，键入make
```
make
```
 - 缺少`aclocal-1.15`
```
...
WARNING: 'aclocal-1.15' is missing on your system. 
You should only need it if you modified 'acinclude.m4' or 'configure.ac' or m4 files included by 'configure.ac'. 
...
```
需要安装`automake`,直接`sudo apt-get install automake`解决。如果装的版本不是`aclocal-1.15`,其他解决办法[看这里](https://stackoverflow.com/questions/33278928/how-to-overcome-aclocal-1-15-is-missing-on-your-system-warning)。
安装后重新`make`就成功了

## 8. 试运行
看看示例程序能不能运行。
### 8.1 创建`www/`文件夹
```
$ pwd 
/home/username/Desktop/mtcp
$ mkdir ../www
```
### 8.2 创建文件
这个比较随意，在`www/`文件夹里随便创建几个文件就行
```
$ touch ../www/index.html
$ history > ../www/history
$ ls ../www/
index.html  history
```
### 8.3 试运行`epserver`
```
$ cd apps/example/
$ sudo ./epserver -p /home/username/Desktop/www -f epserver.conf -N 1
                                        //这里的-N 我输入的是1，因为我虚拟机只有一个核，你们可以根据自己情况修改  
```
只要输出cpu信息就运行成功了，`^C`停止运行

## 9. 配置route和arp
进入项目根目录后，进入`config/`，创建并修改两个配置文件`config/route.conf`、`config/arp.conf`。
```
$ cd config/
$ cp sample_arp.conf arp.conf
$ cp sample_route.conf route.conf
$ ifconfig           //查看并记录网卡MAC地址，
$ vim arp.conf
$ vim route.conf
```
将两个文件按自己需求配置，我的配置如下。由于这次是虚拟机，所以两台虚拟机都这样配置没问题。
```
# arp表
ARP_ENTRY 2
192.168.198.130/32 00:0C:29:5A:68:0E
192.168.198.131/32 00:0C:29:D2:C8:1B
```
```
# route表
ROUTES 2
192.168.198.0/24 dpdk0
10.0.1.1/24 dpdk1    //这一项不管它，我就一个dpdk网卡
```
* 把config/目录拷贝到epserver和epwget所在文件夹
  ```
  $ cd ../
  $ cp -r config/ apps/example/config
  ```

## 10. 正式运行
服务器端和客户端都进入`example/`文件夹
```
$ cd apps/example
```
### 10.1 开启服务器端
```
# 服务器端 IP为192.168.198.131
$ sudo ./epserver -p /home/username/Desktop/www -f epserver.conf -N 1
```
### 10.2 开启客户端
```
# 客户端 IP问为192.168.198.130
$ sudo ./epwget 192.168.198.131/index.html 10000 -N 1 -c 10 -f epwget.conf
                                      //一万个请求先试试水
$ sudo ./epwget 192.168.198.131/index.html 100000 -N 1 -c 50 -f epwget.conf
          //这个index.html就是之前创建的，一共十万请求，并发线程数为50
```
### 10.3 运行结果
```
# 服务器端
$ sudo ./epserver -p /home/username/Desktop/www/ -f epserver.conf -N 1
Configuration updated by mtcp_setconf().
---------------------------------------------------------------------------------
Loading mtcp configuration from : epserver.conf
Loading interface setting
EAL: Detected 1 lcore(s)
EAL: Detected 1 NUMA nodes
EAL: Auto-detected process type: PRIMARY
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: No free hugepages reported in hugepages-1048576kB
EAL: Probing VFIO support...
EAL: PCI device 0000:02:01.0 on NUMA socket -1
EAL:   Invalid NUMA socket, default to 0
EAL:   probe driver: 8086:100f net_e1000_em
EAL: Error reading from file descriptor 11: Input/output error
Total number of attached devices: 1
Interface name: dpdk0
EAL: Auto-detected process type: PRIMARY
Configurations:
Number of CPU cores available: 1
Number of CPU cores to use: 1
Maximum number of concurrency per core: 10000
Maximum number of preallocated buffers per core: 10000
Receive buffer size: 8192
Send buffer size: 8192
TCP timeout seconds: 30
TCP timewait seconds: 0
NICs to print statistics: dpdk0
---------------------------------------------------------------------------------
Interfaces:
name: dpdk0, ifindex: 0, hwaddr: 00:0C:29:D2:C8:1B, ipaddr: 192.168.198.131, netmask: 255.255.255.0
Number of NIC queues: 1
---------------------------------------------------------------------------------
Loading routing configurations from : config/route.conf      //路由表
Routes:
Destination: 192.168.198.0/24, Mask: 255.255.255.0, Masked: 192.168.198.0, Route: ifdx-0
Destination: 192.168.198.131/24, Mask: 255.255.255.0, Masked: 192.168.198.0, Route: ifdx-0
Destination: 10.0.1.1/24, Mask: 255.255.255.0, Masked: 10.0.1.0, Route: ifdx--1
---------------------------------------------------------------------------------
Loading ARP table from : config/arp.conf                      //ARP表
ARP Table:
IP addr: 192.168.198.131, dst_hwaddr: 00:0C:29:D2:C8:1B
IP addr: 192.168.198.130, dst_hwaddr: 00:0C:29:5A:68:0E
---------------------------------------------------------------------------------
Initializing port 0... EAL: Error enabling interrupts for fd 11 (Input/output error)
done: 

Checking link statusdone
Port 0 Link Up - speed 1000 Mbps - full-duplex
CPU 0: initialization finished.
[mtcp_create_context:1268] CPU 0 is now the master thread.
[CPU 0] dpdk0 flows:      0, RX:       4(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[ ALL ] dpdk0 flows:      0, RX:       4(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
.......                  
[CPU 0] dpdk0 flows:      0, RX:       1(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[ ALL ] dpdk0 flows:      0, RX:       1(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[CPU 0] dpdk0 flows:      0, RX:       2(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[ ALL ] dpdk0 flows:      0, RX:       2(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[CPU 0] dpdk0 flows:      0, RX:       6(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[ ALL ] dpdk0 flows:      0, RX:       6(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
......                          // 一万个请求开始
[CPU 0] dpdk0 flows:     81, RX:   16394(pps) (err:     0),  0.02(Gbps), TX:   16141(pps),  0.02(Gbps)
[ ALL ] dpdk0 flows:     81, RX:   16394(pps) (err:     0),  0.02(Gbps), TX:   16141(pps),  0.02(Gbps)
[CPU 0] dpdk0 flows:      0, RX:   34669(pps) (err:     0),  0.03(Gbps), TX:   33859(pps),  0.04(Gbps)
[ ALL ] dpdk0 flows:      0, RX:   34669(pps) (err:     0),  0.03(Gbps), TX:   33859(pps),  0.04(Gbps)
[CPU 0] dpdk0 flows:      0, RX:       1(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[ ALL ] dpdk0 flows:      0, RX:       1(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
......                          // 十万个请求开始
[CPU 0] dpdk0 flows:      0, RX:       0(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[ ALL ] dpdk0 flows:      0, RX:       0(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[CPU 0] dpdk0 flows:     50, RX:   19267(pps) (err:     0),  0.02(Gbps), TX:   18863(pps),  0.02(Gbps)
[ ALL ] dpdk0 flows:     50, RX:   19267(pps) (err:     0),  0.02(Gbps), TX:   18863(pps),  0.02(Gbps)
[CPU 0] dpdk0 flows:     63, RX:  216818(pps) (err:     0),  0.20(Gbps), TX:  212478(pps),  0.23(Gbps)
[ ALL ] dpdk0 flows:     63, RX:  216818(pps) (err:     0),  0.20(Gbps), TX:  212478(pps),  0.23(Gbps)
[CPU 0] dpdk0 flows:     65, RX:  261800(pps) (err:     0),  0.24(Gbps), TX:  256043(pps),  0.28(Gbps)
[ ALL ] dpdk0 flows:     65, RX:  261800(pps) (err:     0),  0.24(Gbps), TX:  256043(pps),  0.28(Gbps)
[CPU 0] dpdk0 flows:      0, RX:   13056(pps) (err:     0),  0.01(Gbps), TX:   12649(pps),  0.01(Gbps)
[ ALL ] dpdk0 flows:      0, RX:   13056(pps) (err:     0),  0.01(Gbps), TX:   12649(pps),  0.01(Gbps)
[CPU 0] dpdk0 flows:      0, RX:       0(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
......
[CPU 0] dpdk0 flows:      0, RX:       0(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[ ALL ] dpdk0 flows:      0, RX:       0(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
^C[RunMainLoop: 868] MTCP thread 0 finished.
[mtcp_free_context:1312] MTCP thread 0 joined.
[mtcp_destroy:1570] All MTCP threads are joined.
```
```
# 客户端
$ sudo ./epwget 192.168.198.131/index.html 100000 -N 1 -c 50 -f epwget.conf 
Configuration updated by mtcp_setconf().
Application configuration:
URL: /index.html
# of total_flows: 100000
# of cores: 1
Concurrency: 50
---------------------------------------------------------------------------------
Loading mtcp configuration from : epwget.conf
Loading interface setting
EAL: Detected 1 lcore(s)
EAL: Detected 1 NUMA nodes
EAL: Auto-detected process type: PRIMARY
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: No free hugepages reported in hugepages-1048576kB
EAL: Probing VFIO support...
EAL: PCI device 0000:02:01.0 on NUMA socket -1
EAL:   Invalid NUMA socket, default to 0
EAL:   probe driver: 8086:100f net_e1000_em
EAL: Error reading from file descriptor 10: Input/output error
Total number of attached devices: 1
Interface name: dpdk0
EAL: Auto-detected process type: PRIMARY
Configurations:
Number of CPU cores available: 1
Number of CPU cores to use: 1
Maximum number of concurrency per core: 10000
Maximum number of preallocated buffers per core: 10000
Receive buffer size: 8192
Send buffer size: 8192
TCP timeout seconds: 30
TCP timewait seconds: 0
NICs to print statistics: dpdk0
---------------------------------------------------------------------------------
Interfaces:
name: dpdk0, ifindex: 0, hwaddr: 00:0C:29:5A:68:0E, ipaddr: 192.168.198.130, netmask: 255.255.255.0
Number of NIC queues: 1
---------------------------------------------------------------------------------
Loading routing configurations from : config/route.conf
Routes:
Destination: 192.168.198.0/24, Mask: 255.255.255.0, Masked: 192.168.198.0, Route: ifdx-0
Destination: 192.168.198.130/24, Mask: 255.255.255.0, Masked: 192.168.198.0, Route: ifdx-0
Destination: 10.0.1.1/24, Mask: 255.255.255.0, Masked: 10.0.1.0, Route: ifdx--1
---------------------------------------------------------------------------------
Loading ARP table from : config/arp.conf
ARP Table:
IP addr: 192.168.198.130, dst_hwaddr: 00:0C:29:5A:68:0E
IP addr: 192.168.198.131, dst_hwaddr: 00:0C:29:D2:C8:1B
---------------------------------------------------------------------------------
Initializing port 0... EAL: Error enabling interrupts for fd 10 (Input/output error)
done: 

Checking link statusdone
Port 0 Link Up - speed 1000 Mbps - full-duplex
Configuration updated by mtcp_setconf().
CPU 0: initialization finished.
[mtcp_create_context:1268] CPU 0 is now the master thread.
[CPU 0] dpdk0 flows:      0, RX:       0(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
[ ALL ] dpdk0 flows:      0, RX:       0(pps) (err:     0),  0.00(Gbps), TX:       0(pps),  0.00(Gbps)
Thread 0 handles 100000 flows. connecting to 192.168.198.131:80
Response size set to 223
[ ALL ] connect:    6130, read:    1 MB, write:    0 MB, completes:    6080 (resp_time avg: 1246, max:  26291 us)
[CPU 0] dpdk0 flows:     89, RX:  192861(pps) (err:     0),  0.21(Gbps), TX:  196842(pps),  0.18(Gbps)
[ ALL ] dpdk0 flows:     89, RX:  192861(pps) (err:     0),  0.21(Gbps), TX:  196842(pps),  0.18(Gbps)
[ ALL ] connect:   45582, read:    9 MB, write:    5 MB, completes:   45582 (resp_time avg:  594, max:  20549 us)
[CPU 0] dpdk0 flows:     85, RX:  245700(pps) (err:     0),  0.27(Gbps), TX:  251219(pps),  0.23(Gbps)
[ ALL ] dpdk0 flows:     85, RX:  245700(pps) (err:     0),  0.27(Gbps), TX:  251219(pps),  0.23(Gbps)
[CPU 0] Completed 100000 connections, errors: 0 incompletes: 0
[RunMainLoop: 868] MTCP thread 0 finished.
[mtcp_free_context:1312] MTCP thread 0 joined.
[mtcp_destroy:1570] All MTCP threads are joined.
```

## 11. 恢复原状
仍然先回到项目根目录。执行
```
$ ./setup_mtcp_dpdk_env.sh /home/username/Desktop/mtcp/dpdk
```
```
# 选项
[30] Unbind devices from IGB UIO or VFIO driver
[31] Remove IGB UIO module
[34] Remove hugepage mappings
[35] Exit Script
```
- [30]解绑网卡和IGB驱动时，会先要求你输入dpdk网卡的PCI地址，然后让你输入新的驱动，给出网卡信息后，`unused=`后面会有可使用的驱动，我的是`e1000`，输入`e1000`就行。
- [35]退出时可以选no

恢复配置后可以重启网络服务，会自动获取IP地址
```
$ sudo servie network-manager restart
```


