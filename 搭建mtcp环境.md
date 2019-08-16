# mtcp_learning
在计算所网络组实习。第一天，搭建mtcp环境。

## 0. 环境介绍
我使用的是所里提供的服务器。
```
uname -a
Linux localhost.localdomain 3.10.0-862.11.6.el7.x86_64 #1 SMP Tue Aug 14 21:49:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

## 1. clone原项目代码
```
git clone https://github.com/mtcp-stack/mtcp
```

 * 不同版本的mtcp环境搭建步骤不同，请注意clone的版本。 我使用的版本是2019.3.23更新的。
 
## 2. 预装库
所里服务器上已经把大部分库都装好了。这一步我没有执行。

## 3. 按照README执行命令
 * 下载DPDK
 ```
 git submodule init
 git submodule update
 ```
 
 * 安装DPDK
 ```
 ./setup_mtcp_dpdk_env.sh [<path to $RTE_SDK>]  // RTE_SDK就是dpdk的安装目录
                                            //就是clone下来的mtcp/dpdk文件夹，初始为空
 ```
 **这一步巨坑，path一定要用绝对路径！！！**
 
 比如我的执行的就是
 ```
 ./setup_mtcp_dpdk_env.sh /home/username/mtcp/dpdk
 ```
 接着往下执行
```
 Press [15] to compile x86_64-native-linuxapp-gcc version
 Press [18] to install igb_uio driver for Intel NICs
 Press [22] to setup 2048 2MB hugepages
 Press [24] to register the Ethernet ports
 Press [35] to quit the tool
```
 在第24项选择网卡时，一定要选择未使用的网卡。输入的PCI地址`0000:02:00.3`类似这种格式。[如何获取网卡的PCI地址](https://askubuntu.com/questions/654820/how-to-find-pci-address-of-an-ethernet-interface)
 
 接着可以查看用`ifconfig -a`查看网卡，会多一个dpdk0网卡。
 
 **编译的时候也会报错！！！如果编译的时候netdevice.h报错，可能是结构体变量名没有和内核保持一致。**
 
 找到报错的文件，根据内核定义的结构体修改变量名。我的修改如下：
 
    ndo_change_mtu -> ndo_change_mturh74
    ndo_set_vf_vlan -> ndo_set_vf_vlan_rh73
    ndo_setup_tc -> ndo_setup_tc_rh72
 
  * 启用网卡
    ```bash
    sudo ifconfig dpdk0 x.x.x.x netmask 255.255.255.0 up
    export RTE_SDK=`echo $PWD`/dpdk   
    export RTE_TARGET=x86_64-native-linuxapp-gcc
     ```
 
  *  装mtcp库
  
    ```bash
    ./configure --with-dpdk-lib=$RTE_SDK/$RTE_TARGET
   	make
    ```
    
 在执行`./configure`的时候可能会报错。我这里报了一个找不到`gmp.h`
 
 ```
 ....
 configure: error: Could not find gmp.h
 ```
 
 缺少库文件，执行如下命令
 ```
 yum install -y gmp-devel
 ```
 
 重新执行`./configure --with……`
 然后执行`make`，提示成功。
 
 
 
 
 
 
 
