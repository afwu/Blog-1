---
title: Moloch安装与使用
date: 2020-04-10 21:21:21
tags: threathunting
---

## 0X01 moloch简介

Moloch是一款由 AOL 开源的，能够大规模的捕获IPv4数据包（PCAP）、索引和数据库系统。所以我的Capture Machines和Elasticsearch Machines都放在一台上面， 有条件的强烈推荐把这2个组件分离开来。

根据官方介绍，需要留意一下事情：

- 1.Moloch不支持32位
- 2.内核4.X 有助于抓包性能提升

## 0x02 Moloch安装

### 1.环境配置

```bash
[admin@localhost ~]$ uname -a
Linux localhost.localdomain 4.18.0-151.el8.x86_64 #1 SMP Wed Dec 4 17:04:30 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

### 2.Moloch下载

[官网即可下载](https://molo.ch/downloads)根据自己的配置环境下载对应的版本的安装包。

### 3.Moloch安装

安装程序之前需要先安装三个依赖包

```bash
[root@localhost admin]# yum install -y perl-libwww-perl perl-JSON libyaml-devel
```

cnetos8在安装libyaml-devel时会提示找不到依赖库，建议采用以下方法安装。

```bash
[root@192 admin]# wget http://mirror.centos.org/centos/8/PowerTools/x86_64/os/Packages/libyaml-devel-0.1.7-5.el8.x86_64.rpm
[root@192 admin]# rpm -ivh libyaml-devel-0.1.7-5.el8.x86_64.rpm
```

依赖包安装成功后，可以继续安装moloch。依赖包libyaml-devel[下载地址](https://pkgs.org/download/libyaml-devel)

```bash
[root@192 admin]# rpm -ivh moloch-2.2.2-1.x86_64.rpm
```

### 4.安装pfring

moloch的Capture默认使用libpcap后面我们可以用pfring 提升抓包性能。

```bash
[root@192 admin]#  cd /etc/yum.repos.d/
[root@192 yum.repos.d]# wget http://packages.ntop.org/centos-stable/ntop.repo -O ntop.repo• wget http://packages.ntop.org/centos-stable/epel-7.repo -O epel.repo•   yum erase zeromq3 #添加pf_ring官方的yum源，系统自带的yum源没有pf_ring的yum包。
[root@192 yum.repos.d]# yum clean all
[root@192 yum.repos.d]# yum update
[root@192 yum.repos.d]# yum install pfring
```

## 0x03.elasticsearch安装配置

es可以选择在配置moloch时候安装 也可以自己单独安装 我选择自己单独安装。

Es的下载地址：<https://www.elastic.co/downloads/elasticsearch>

这里使用的是：<https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.1.rpm>

```bash
[root@192 aaa]# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.1.rpm
[root@192 aaa]# rpm -ivh elasticsearch-6.4.1.rpm #需要提前预置java环境
```

安装完成之后，修改相关配置

```bash
[root@192 /]# cd etc/
[root@192 etc]# cd elasticsearch/
```

### 1.设置虚拟内存

```bash
[root@192 elasticsearch]# vim jvm.options
# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space

-Xms1g #设置一下虚拟内存，如果内存够大就多设置，小就少设置。
-Xmx1g
```

### 2.设置存储告警

```bash
[root@192 elasticsearch]# vim elasticsearch.yml

cluster.routing.allocation.disk.threshold_enabled: falsenetwork.host: 0.0.0.0 #抓包经常会把硬盘用完，当硬盘使用空间到80%。es就开始报警，直接把报警关掉。

#
#network.host: 0.0.0.0 #这里建议 0.0.0.0
#
# Set a custom port for HTTP:
#
#http.port: 9200
```

### 3.系统设置修改

```bash
#第一步：修改/etc/security/limits.conf文件，增加配置，用户退出后重新登录生效
*                soft    nproc           65535
*                hard    nproc           65535
*                soft    nofile          65535
*                hard    nofile          65535

#第二步：因为上方是针对所有用户的设置，这样会受系统限制，咱们还需要更改一下系统限制。
#修改/etc/security/limits.d/20-nproc.conf，没有这个文件的可以直接创建
*          soft    nproc     65535
root       soft    nproc     unlimited

#第三步：还需要修改虚拟内存大小，修改/etc/sysctl.conf文件在最后面追加下面内容 会永久生效
vm.max_map_count=262144
kernel.pid_max = 65535  #这个可填可不填

#第四步：使用 如下命令 第一个查看修改后的结果，第二个可多开文件数
sysctl -p
ulimit -n 65536

#第五步：防火墙永久增加端口让其他机器可以访问
#添加
[root@192 /]# firewall-cmd --zone=public --add-port=9200/tcp --permanent
success
[root@192 /]# firewall-cmd --zone=public --add-port=8005/tcp --permanent
success
[root@192 /]# firewall-cmd --reload #重新载入
success
```

### 4.查看es状态

```bash
[root@192 /]# systemctl start elasticsearch.service  #启动ES

[root@192 /]# systemctl status elasticsearch.service #查看ES启动状态
● elasticsearch.service - Elasticsearch #启动成功，前边为绿色
   Loaded: loaded (/usr/lib/systemd/system/elasticsearch.service; disabled; ven>
   Active: active (running) since Wed 2020-04-08 06:09:13 EDT; 6s ago
     Docs: http://www.elastic.co
 Main PID: 1625 (java)
    Tasks: 4
   Memory: 356.8M
   CGroup: /system.slice/elasticsearch.service
           └─1625 /bin/java -Xms1g -Xmx1g -XX:+UseConcMarkSweepGC -XX:CMSInitia>

Apr 08 06:09:13 192.168.126.155 systemd[1]: Started Elasticsearch.
```

最后访问一下：<http://127.0.0.1:9200>，出现以下信息证明安装没有问题。

```json
{
  "name" : "JskrJLn",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "CUduIznDSPiIhyF9fssK7g",
  "version" : {
    "number" : "6.4.1",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "e36acdb",
    "build_date" : "2018-09-13T22:18:07.696808Z",
    "build_snapshot" : false,
    "lucene_version" : "7.4.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

## 0x04.moloch设置

进行安装后的设置目录。运行./configure

```bash

[root@192 /]# cd /data/moloch/bin/
[root@192 bin]# ./Configure
Found interfaces: ens33;lo;virbr0
Semicolon ';' seperated list of interfaces to monitor [eth1] ens33 #输入你自己对应的网卡
Install Elasticsearch server locally for demo, must have at least 3G of memory,                                                                                                                                NOT recommended for production use (yes or no) [no] no #输入no即可
Elasticsearch server URL [http://localhost:9200] http://127.0.0.1:9200 #elasticsearch服务器地址  一般默认本机或者你指定的地址  必须以http://开头
Password to encrypt S2S and other things [no-default] root #默认的登陆密码,可自行设置
Moloch - Creating configuration files
Installing systemd start files, use systemctl
Moloch - Installing /etc/logrotate.d/moloch to rotate files after 7 days
Moloch - Installing /etc/security/limits.d/99-moloch.conf to make core and memlo                                                                                                                               ck unlimited
Download GEO files? (yes or no) [yes] yes #下载一些文件。直接“yes”即可
```

具体关于moloch设置信息详情参考官网，也可参考<https://woj.app/3872.html>

### 1.升级初始化

```bash
/data/moloch/db/db.pl  http://localhost:9200  init    ##第一次安装初始化、或者想删除所有数据
/data/moloch/db/db.pl  http://localhost:9200 upgrade  ##升级moloch 数据包
#记得输入UPGRADE
/data/moloch/bin/moloch_add_user.sh admin "Admin User" moloch --admin  ##新增admin账户，密码是moloch
systemctl enable molochcapture.service ##开机启动Capture
systemctl start molochcapture.service  ##启动Capture
systemctl enable molochviewer.service  ##开机启动Viewer
systemctl start molochviewer.service   ##启动Viewer
```

**如果不是特别讲究的话，操作到这里就可以了。**

### 2.数据删除

由于是虚拟机部署的，所以需要及时清理数据，否则虚拟机很快会因为数据过大，卡顿。

```bash
# 关于pcap的数据包，可使用moloch来控制删除
[root@moloch ~]# vim /data/moloch/etc/config.ini #不同版本的moloch，路径也不同
# moloch 默认是freeSpaceG = 5%，也就是磁盘空间会保留5%。
freeSpaceG = 5%
```

```bash
# es使用moloch自带的脚本来控制删除
[root@moloch db]# vim daily.sh
#!/bin/sh
# This script is only needed for Moloch deployments that monitor live traffic.
# It drops the old index and optimizes yesterdays index.
# It should be run once a day during non peak time.
# CONFIG
ESHOSTPORT=10.100.10.7:9200
RETAINNUMDAYS=1
/data/moloch/db/db.pl $ESHOSTPORT expire daily $RETAINNUMDAYS
```

```bash
# 再做个定时任务 每天晚上跑一次
[root@moloch ~]# crontab -e
01 04 * * * /data/moloch/db/daily.sh >> /var/log/moloch/daily.log 2>&1
```

### 3.网卡优化

```bash
# Set ring buf size, see max with ethool -g eth0
ethtool -G eth0 rx 4096 tx 4096
# Turn off feature, see available features with ethtool -k eth0
ethtool -K eth0 rx off tx off gs off tso off gso off
```

### 4.High Performance Settings 高性能设置

```bash
# MOST IMPORTANT, use basic magicMode, libfile kills performance
magicMode=basic

# 官方称pfring效果更好
# pfring/snf might be better
pcapReadMethod=tpacketv3

# Increase by 1 if still getting Input Drops
tpacketv3NumThreads=2

# Defaults
pcapWriteMethod=simple
pcapWriteSize = 2560000

# Start with 5 packet threads, increase by 1 if getting thread drops
packetThreads=5

# Set to number of packets a second
maxPacketsInQueue = 200000
```

### 5.pfring 配置

```bash
[root@moloch ~]# vim /data/moloch/etc/config.ini
rootPlugins=reader-pfring.so
pcapReadMethod=pfring
```

## 0x05.moloch使用

作为二线分析人员，大部分情况下不需要进行实时抓包分析流量的，那么我们只需要导入数据包进行分析即可。需要把moloch抓包功能关闭掉。

```bash
#导入离线pcap文件，如果遇到提示GEO存在问题的，建议重新初始化即可
cd /data/moloch/bin
./capture/moloch-capture –R /offline-pcap-dir <--recursive> <--copy>

./moloch-capture -r /data/moloch/raw/infile.pcap

-r #pcapfile脱机pcap文件
-R #pcapdir脱机pcap目录，将处理所有*.pcap文件
--recursive #可以遍历子目录
--copy #可以将pcap文件拷贝到config.ini中指定的pcapDir


#删除pcap文件
./moloch-capture --copy --delete /home/jx/test.pcapng #路径
```

```bash
#网页查看导入数据
file == "/home/admin/http.pcap" #导入数据路径
```

## 0x06.总结

记录一下本文中提到的服务启动命令吧。

```bash
systemctl start elasticsearch.service  #es启动命令，访问127.0.0.1:9200即可
systemctl start molochviewer.service molochcapture.service #moloch启动命令、访问IP:8005即可，密码是自己设置的。
```

添加用户

```bash
/data/moloch/bin/moloch_add_user.sh admin "Admin User" moloch --admin  ##新增admin账户，密码是moloch
```

## 0x07.参考推荐

Moloch快速安装部署:<https://www.cnblogs.com/yaoyaojcy/p/11727122.html>

Moloch优化及安装部署:<https://woj.app/3872.html/comment-page-1>

Moloch配置及汉化说明:<https://woj.app/4008.html>

Moloch-zh:<https://github.com/xazlsec/Moloch-zh>
