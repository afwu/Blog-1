---
title: 基于流量侧检测冰蝎webshell交互通讯
date: 2020-04-18 21:21:21
tags: 冰蝎
---

## 简述

作为新型加密webshell管理客户端，冰蝎算是作为中国菜刀、C刀的替代者。根据网传使用效果，基本得到的反馈是相当的NICE！那么我们能否像检测中国菜刀、C刀那样对冰蝎客户端的流量进行检测，帮助网站管理员判断自己的网站是否存在后门。本文后续一一介绍。

## 一、冰蝎主要功能

### 1.基本信息

客户端和服务端握手之后，会获取服务器的基本信息，Java、.NET版本包括环境变量、系统属性等，PHP版本会显示phpinfo的内容。

### 2.文件管理

和中国菜刀、C刀的功能差不多，文件的增删改查，稍微不同的是文件都是进行了加密传输的，可以避免被拦截。

### 3.命令执行

执行单条操作系统命令。

### 4.虚拟终端

虚拟终端提供了一个模拟真实的交互式shell环境，相当于把服务器侧的shell客户端放在了管理工具内，在这个shell里你可以执行各种需要交互的命令，当然你也可以利用它对内网进行攻击测试。

### 5.Scoks代理

虚拟终端功能其实已经是部分实现了内网穿透的能力。在shell环境里所进行的操作其实都是在内网环境下进行的，不过为了方便使用，客户端提供了基于一句话木马的Socks代理功能，一键开启，简单高效。

### 6.反弹shell

反弹Shell是突破防火墙的利器，也几乎是后渗透过程的必备步骤。提到后渗透，当然少不了metasploit，提到metasploit，当然少不了meterpreter，所以冰蝎客户端提供了两种反弹Shell的方式，常规Shell和Meterpreter，实现和metasploit的一键无缝对接。

### 7.数据库管理

常规功能，实现了数据库的可视化管理，和常规管理工具不同的是，在Java和.NET环境中，当目标机器中没有对应数据库的驱动时，会自动上传并加载数据库驱动。比如目标程序用的是MySQL的数据，但是内网有另外一台Oracle，此时就会自动上传并加载Oracle对应的驱动。

### 8.自定义代码

可以在服务端执行任意的Java、PHP、C#代码，这也是个常规功能，值得一提的是我们输入的代码都是加密传输的，所以不用为了躲避waf而用各种编码变形。

### 9.备忘录

渗透的时候总有很多零碎的信息需要记录，所以针对每个Shell提供了一个备忘录的功能，目前只支持纯文本，粘贴进去自动保存。

## 二、基于流量检测

基于流量检测，自然需要进行模拟测试。如果你有兴趣，你也可以进行本地模拟测试，把正常的网站访问流量和冰蝎交互通讯流量放在一起，进行对比，很快，你也能够找到它们的不同之处。

### 1.测试环境

PHP网站（Phpstudy+DVWA）

冰蝎V1.0/V2.0（客户端+自带shell）

wireshark（必备）

### 2.流量对比

#### 冰蝎V1.0

##### 1）正常流量

![01](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pLmxvbGkubmV0LzIwMTkvMDgvMjAvQ2dzbHZMY2lyNXpwOURWLnBuZw)

##### 冰蝎客户端与服务端通信流量

![03.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pLmxvbGkubmV0LzIwMTkvMDgvMjAvOElIaTV1REx4Zk56Y0FsLnBuZw)

##### 正常流量VS冰蝎流量

![user-agent.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pLmxvbGkubmV0LzIwMTkvMDgvMjAvcTR3U2FNWmJKY0xBdURRLnBuZw)

从中可以看到冰蝎V1.0版本初期交互时，特征较为明显，user-agent与正常业务流量明显不同。可以通过对**user-agent**进行检测分析。其次在POST返回包中相对正常流量多了**Transfer-Encoding: chunked**，Transfer-Encoding主要是用来改变报文格式，这里指的是利用分块进行传输。你可以基于此特征值进行检测，当然，你也可以用更简单的方法进行检测，比如url中包含***.php?pass=**来进行检测。

#### 冰蝎V2.1

从冰蝎V1.1开始新增随机UserAgent支持，每次会话会从17种常见UserAgent中随机选取。冰蝎最新版本为V2.1，可以通过对2.1版本服务端与客户端的通信流量，进行捕获，对比正常流量进行分析。

##### 正常流量

![10.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pLmxvbGkubmV0LzIwMTkvMDgvMjAvNm5lcklaTXdoQWZXdDlwLnBuZw)

##### 冰蝎客户端与服务端流量

![Transfer-Encoding.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pLmxvbGkubmV0LzIwMTkvMDgvMjAveEVuVGxwVVB1NXFaM05ILnBuZw)

##### 正常流量VS冰蝎通讯流量

![VS.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pLmxvbGkubmV0LzIwMTkvMDgvMjAvaUUzN0hnSnZWcGJMd3luLnBuZw)

通过对比可以看到，冰蝎V2.1版本在初期交互通讯时流量中多了一个**Transfer-Encoding: chunked**，Transfer-Encoding主要是用来改变报文格式，这里指的是利用分块进行传输。你可以基于此特征值进行检测，当然，你也可以用更简单的方法进行检测，比如url中包含***.php?pass=**来进行检测。

**值得注意的是，文中提到的检测特征需要特定的环境！**

### 检测特征

#### 冰蝎v1.0特征

基于GET请求包的检测特征：url包含.php?pass=，useragent包含Java/*；

基于POST请求包的检测特征：useragent包含Java/* ，返回包包含：Transfer-Encoding: chunked；

#### 冰蝎V2.1特征

基于GET请求包的检测特征：url包含.php?pass=；（如果与业务冲突，误报较大）

基于POST返回包的检测特征：Transfer-Encoding: chunked；

关于详细的你也可以查看[T1100-weshell-冰蝎检测](https://github.com/12306Bro/My-life/blob/master/T1100-webshell-%E5%86%B0%E8%9D%8E.md)

## 三、厂商检测能力

冰蝎真的有大家说的那么强么？真的是无懈可击么？其实安全厂商在某次活动之后就关注了它，并提供了检测规则。以下资料来源于互联网。如有侵权，请及时与我联系！

### 1.绿盟

2019-06-14 绿盟发布日志数据安全性分析系统(BSA)升级包，新增对冰蝎webshell上传检测规则；

参考链接：<http://update.nsfocus.com/update/listBsaUtsDetail/v/rule2.0.0>

2019-06-17  绿盟发布WEB应用防护系统(WAF) 规则升级包，添加检测冰蝎webshell特征；

参考链接：<http://update.nsfocus.com/update/listWafV65Detail/v/rule6.0.5>

### 2.新华三

2019-06-29 新华三发布IPS特征库升级包，新增尝试上传冰蝎webshell后门文件检测规则；

参考链接：<http://www.h3c.com/cn/d_201907/1210668_30003_0.htm>

### 3.百度安全

2019-08-08 百度安全发布《那些年我们堵住的洞—OpenRASP纪实》一文，文中提到可对冰蝎动态后门JSP版进行检测；

参考链接：<https://anquan.baidu.com/article/855>

### 4.其他待补充

暂无
