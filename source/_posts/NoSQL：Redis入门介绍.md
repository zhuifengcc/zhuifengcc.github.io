---
title: 'NoSQL：Redis入门介绍'
date: 2018-06-05 17:07:44
tags:
categories: NoSql
---
## 一、Redis入门概述
### 1.Redis是什么
Redis：REmote DIctionary Server（远程字典服务器），是一个高性能的（key/value）分布式内存数据库，基于内存运行，并支持持久化的NOSQL数据库，也被人们称为数据结构数据库。

    Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用
    Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset（sorted set），hash等数据结构的存储
    Redis支持数据的备份，即master-slave模式的数据备份
### 2.Redis的安装
我们在虚拟机上以centos为例进行Redis的安装：

![登录页面](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-1.png)

因为许久没有碰linux了，这里将整体过程及踩到的坑做一个记录：

默认安装好系统后时无法使用wget的，本打算用yum下载wget，再用wget完成对redis的下载，如下所示：

![直接运行wget页面](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-2.png)

![yum下载wget页面](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-3.png)

这是因为网络是不通的，需要配置网络，因为我使用的是笔记本的无线网络，虚拟机也需要同步使用，连接方式使用NAT尝试未成功，所以最后我将VMware连接方式调整成了桥接

![ping尝试](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-4.png)

![桥接配置](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-5.png)

![桥接配置](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-6.png)

查看物理机ip信息
![查看物理机ip](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-7.png)

根据物理主机的ip地址，设置linux虚拟机的ip地址:

方法：修改/etc/sysconfig/network-scripts/ifcfg-eth0这个配置文件

    vim /etc/sysconfig/network-scripts/ifcfg-eth0
    DEVICE=eth0    #虚拟机网卡名称。
    TYPE=Ethernet
    ONBOOT=yes　　   #开机启用网络配置。
    NM_CONTROLLED=yes
    BOOTPROTO=static   #static，静态ip，而不是dhcp，自动获取ip地址。
    IPADDR=192.168.1.109　　#设置我想用的静态ip地址，要和物理主机在同一网段，但又不能相同。
    NETMASK=255.255.255.0 #子网掩码，和物理主机一样就可以了。
    GETWAY=192.168.1.1  #和物理主机一样
    DNS1=8.8.8.8　　　　　　#DNS，写谷歌的地址就可以了。
    HWADDR=00:0c:29:22:05:4c
    IPV6INIT=no
    USERCTL=no  
在netwotk-scripts文件夹下，如果并没有所说的ifcfg-eth0这个文件，而是ifcfg-ens33还有ifcfg-lo, 通过vi命令编辑ens33这个文件，最后把文件改名就ok了, 必须有ifcfg-eth0这个配置文件
![网络配置目录](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-8.png)

![网络配置](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-9.png)

重命名：mv ifcfg-ens33 ifcfg-eth0

![网络配置](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-10.png)

如果你没有ifcfg-eth,是改ens33的，还需要下面俩步

用vi命令编辑/etc/default/grub 并在GRUBCMDLINELINUX变量加入“net.ifnames=0 biosdevname=0 ”，如下

![网络配置](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-11.png)

修改完之后，按ESC，然后输入 ：wq（加！表示保存只读文件的修改）退出

运行命令grub2-mkconfig -o /boot/grub2/grub.cfg 来重新生成GRUB配置并更新内核参数。

最后重启 reboot

重启后可以尝试ping外网，发现此时网络可以连通了，于是可以通过yum下载wget，然后通过wget下载redis了，将redis下载到/opt下

![尝试ping](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-12.png)

![yum下载wget](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-13.png)

![yum下载wget](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-14.png)

![wget下载redis](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-15.png)

![wget下载redis](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-16.png)

![redis解压](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-17.png)

下载完毕并解压后，进入/opt/redis-4.0.8目录，可以看到redis.conf配置文件，我们将文件拷贝一份至根目录新建的文件夹myredis下以做备份，使用vi命令(less -MN)打开备份的redis.conf文件，在general部分可以看到如下配置，默认redis不会以进程的方式存在，为了后续测试，我们将该属性修改为yes并保存退出

![配置文件备份](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-18.png)

![配置文件备份](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-19.png)

![修改配置](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-20.png)

回到我们解压后的redis的目录，我们对redis进行安装，可以看到目录下有MAKEFILE文件，运行make命令进行编译

![make编译](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-21.png)

![make编译](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-22.png)

此时可以看到运行出错，gcc command not find, gcc是用以编译c/c++项目，与java中的javac类似，这个gcc东东貌似在学校的时候碰到过，好啦，那就是我们目前系统中缺少gcc，我们yum安装它即可

![yum下载gcc](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-23.png)

![gcc安装完成](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-24.png)

安装完毕后在我们解压的redis目录下再次make

![再次make编译](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-25.png)

![再次make编译](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-26.png)

可以看到又报错了，这是搞事情啊，原因在于上次make之后我们的相关目录没有clean干净，可以通过运行make distclean 解决

![make distclean](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-27.png)

![再次make](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-28.png)

执行完毕后我们再次make，可以看到这一次正常make完成了，并且redis给我们了一个提示："It's a good idea to run 'make test'"

![make编译成功](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-29.png)

最后还需要执行make install完成最后的安装

![make install](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-30.png)

![安装完成](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-31.png)

至此，redis的安装就完成了

在默认安装目录/usr/local/bin下我们可以看到生成了下列的文件,可以运行我们备份修改的conf文件使得redis以进程的形式跑起来，同时也可以查看redis相关的进程开启情况，以及通过执行redis-brenchmark来查看部署redis的机器的相关情况，如最大支持的读写数等

![安装目录附图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-32.png)

![运行redis进程附图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-33.png)

执行ps -ef|grep redis 查看redis进程的运行前后对比情况

![进程查看附图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/NoSQL/redis_1/2-34.png)

以上就是redis整个的安装过程，后续对redis的数据结构进行整合和记录
### 3.Redis的配置文件

    1.Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
      daemonize no
    2.当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid，可以通过pidfile指定
      pidfile /var/run/redis.pid
    3.指定Redis监听端口，默认端口为6379(MERZ)
      port 6379
    4.绑定的主机地址
      bind 127.0.0.1
    5.当客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
      timeout 300
    6.指定日志记录级别，Redis总共支持四个级别: debug, verbose, notice, warning，默认为verbose
      loglevel verbose
    7.日志记录方式，默认为标准输出，如果配置Redis为守护进程方式运行，而这里又配置为日志记录方式为标准输出，则日志将会发送给/dev/null
      logfile stdout
    8.设置数据库的数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id
      databases 16
    9.指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
      save <seconds> <changes>
      Redis 默认配置文件中提供了三个条件
      save 900 1  900秒（15分钟）内有1个更改
      save 300 10  300秒（5分钟）内有10个更改
      save 60 10000  60秒内有10000个更改
    10.指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，可以关闭该选项，但会导致数据库文件变的巨大
      rdbcompression yes
    11.指定本地数据库文件名，默认值为dump.rdb
      dbfilename dump.rdb
    12.指定本地数据库存放目录
      dir ./
    13.设置当本机为slave服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步
      slaveof <masterip> <masterport>
    14.当master服务设置了密码保护时，slave服务连接master的密码
      masterauth <master-password>
    15.设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过AUTH <password>命令提供密码，默认关闭
      requirepass 123
    16.设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，如果设置maxclients 0，表示不作限制。当客户端连接数达到限制时，Redis会关闭新的连接并向客户返回 max number of clients reached错误信息
      maxclients 128
