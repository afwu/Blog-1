---
title: windows认证/白银票据检测
date: 2020-04-07 14:58:29
tags: threathunting
---

# 0x01 前言

&ensp;&ensp;&ensp;&ensp;因为疫情导致在家办公，正好可以利用这段时间提升一下自己，所以打算学习一下Windows认证。

# 0x02 windows认证

&ensp;&ensp;&ensp;&ensp;Kerberos是一种网络认证协议，其设计目标是通过密钥系统为客户机/服务器应用程序提供强大的认证服务。该认证过程的实现不依赖于主机操作系统的认证，无需基于主机地址的信任，不要求网络上所有主机的物理安全，并假定网络上传送的数据包可以被任意地读取、修改和插入数据。在以上情况下，Kerberos作为一种可信任的第三方认证服务，是通过传统的密码技术（如：共享密钥）执行认证服务的。<br>

&ensp;&ensp;&ensp;&ensp;整个认证过程需要以下设备：

&ensp;&ensp;&ensp;&ensp;客户端（Client）

&ensp;&ensp;&ensp;&ensp;服务端（Server）

&ensp;&ensp;&ensp;&ensp;认证端（KDC）kerberos认证服务器称KDC，它是由Authentication Service和Ticket Granting Service组成，但是它会访问AD数据库，在认证中会需要到。

 ![认证流程](https://img-blog.csdnimg.cn/20200313125218659.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2MzM0NDY0,size_16,color_FFFFFF,t_70)

## 1.客户端向AS请求

&ensp;&ensp;&ensp;&ensp;首先client会先向DC(KDC>AS)发送请求，但是为了保证传输安全，客户端会使用已登录用户（user1）的Hash加密请求包中的timestamp（时间戳）部分，而域中所有用户的信息（包括hash）都保存在AD（NTDS.DIT）中，这样就只有DC和客户端可以对请求包进行解密，请求包中需要包含的内容有：

&ensp;&ensp;&ensp;&ensp;发送内容① ：[Pre-authentication data（client is ntlm_hash for Timestamp）,Client name & realm（DomainName\Username）,Server Name（KDC TGS NAME）]

&ensp;&ensp;&ensp;&ensp;Pre-authentication data：被客户端加密的timestamp内容

&ensp;&ensp;&ensp;&ensp;Client name & realm:客户端用户名，如user1/h0r2yc.cn

&ensp;&ensp;&ensp;&ensp;Server Name：KDC、TGS的Server Name。

## 2.AS认证通过后发送TGT给客户端

&ensp;&ensp;&ensp;&ensp;DC(KDC>AS)接收到client发来的请求包后，会根据请求包中的用户名（user1）向AD请求user1对应的hash，然后用hash解密出timestamp，如果解密出的timestamp和当前时间未超过5min，验证通过，KDC会生成一个使用user1的hash加密的logon session key，以及一个用krbtgt（用于 Kerberos 身份验证的账户）hash加密的TGT，然后发送给客户端，TGT的有效期为8个小时。

&ensp;&ensp;&ensp;&ensp;发送内容②：[Client_ntlm_hash(K(c,tgs))],[Krbtgt_ntlm_hash(k(c,tgs),Client_name(DomainName\Username),TGT_EndTime)]

&ensp;&ensp;&ensp;&ensp;Logon Session Key：会话凭证

&ensp;&ensp;&ensp;&ensp;Client name & realm：客户端用户，如user1

&ensp;&ensp;&ensp;&ensp;End time: TGT到期的时间

&ensp;&ensp;&ensp;&ensp;客户端接收到数据，使用user1 hash解密出logon session key，但是TGT使用krbtgt hash加密，客户端没有对应的hash所以没办法解密。有了logon session key就可以进行第三步。

&ensp;&ensp;&ensp;&ensp;TGT里面包含PAC，PAC包含Client的sid，Client所在的组。注释：PAC的全称是Privilege Attribute Certificate（特权属性证书）。不同的账号有不同的权限，PAC就是为了区别不同权限的一种方式。

&ensp;&ensp;&ensp;&ensp;大多数服务不验证PAC（通过将PAC校验和发送到域控制器进行PAC验证），因此使用服务帐户密码哈希生成的有效TGS可以完全伪造PAC。

## 3.客户端向TGS发送TGS请求包

&ensp;&ensp;&ensp;&ensp;client接收到DC(KDC>AS)发送回来的数据后，client拿着自己加密的Session_key和TGT凭证向票据生成服务器（TGS）发起一个认证请求（KRB_TGS_REQ）。

&ensp;&ensp;&ensp;&ensp;发送内容③ :[Session_key(Authenticator（[DomainName\Username,ServerName(DomainName\Server)]）)],[TGT]

&ensp;&ensp;&ensp;&ensp;TGT：通过向AS请求获得的TGT

&ensp;&ensp;&ensp;&ensp;Authenticator：客户端使用logon session key加密的联络暗号

&ensp;&ensp;&ensp;&ensp;Client name & realm:客户端用户，如user1

&ensp;&ensp;&ensp;&ensp;Server name & realm:客户端要访问的服务端的用户名

&ensp;&ensp;&ensp;&ensp;Pre-authentication data：被客户端加密的timestamp内容

## 4.TGS认证通过后发送Ticket以及session key

&ensp;&ensp;&ensp;&ensp;TGS接收到client发来的请求后，使用krbtgt hash解密TGT得到logon session key，在使用logon session key解密Authenticator得到联络暗号，认证通过。认证通过后TGS生成使用logon session key加密的session key（session key用于客户端和server进行通讯），再生成一个使用user2 hash加密的ticket，然后发送给客户端。

&ensp;&ensp;&ensp;&ensp;发送内容④：k(c,tgs)加密[Session_key],[Server_ntlm_hash(Tiket（K(c,s),Client_Name(domainName\Username),TGT_EndTime）)]

&ensp;&ensp;&ensp;&ensp;Session Key：用于客户端和server通讯的key。

&ensp;&ensp;&ensp;&ensp;Client name & realm: 客户端用户，如user1

&ensp;&ensp;&ensp;&ensp;End time: Ticket的到期时间

## 5.客户端向server发送请求

&ensp;&ensp;&ensp;&ensp;client接收到数据后，使用logon session key解密出session key，有了session key和Ticket后，就可以直接与server进行交互，无需再通过KDC认证。这时候client创建一个使用session key加密的Authenticator和timestamp（时间戳），然后client将加密过的timestamp、Authenticator以及ticket，还有一个询问是否需要双向验证的flag发送给server。

&ensp;&ensp;&ensp;&ensp;发送内容⑤：K(c,s)加密[Authenticator（[DomainName\Username,ServerName(DomainName\Server)]）],[Tiket]

## 6.server验证通过允许访问

&ensp;&ensp;&ensp;&ensp;server接收到请求包后，使用user2 hash解密Ticket得到session key，再使用session key解密Authenticator和timestamp，如果timestamp和当前时间不超过5min，验证通过，允许访问。

&ensp;&ensp;&ensp;&ensp;发送内容⑥：K(c,s)加密[Authenticator]

# 0x03 白银票据分析及利用

&ensp;&ensp;&ensp;&ensp;白银票据是一个有效的票据授予服务（TGS）Kerberos票据，因为Kerberos验证服务运行的每台服务器都对服务主体名称的服务帐户进行加密和签名。

&ensp;&ensp;&ensp;&ensp;白银票据主要是发生在第五步骤上。

## 1.首先我们尝试访问目标服务器根目录
![dir](https://img-blog.csdnimg.cn/20200313125239529.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2MzM0NDY0,size_16,color_FFFFFF,t_70)

&ensp;&ensp;&ensp;&ensp;访问主域控文件共享，可以看到是无法访问的，说明当前用户密码存在问题，也可以理解为无权限。

## 2.抓取hash

&ensp;&ensp;&ensp;&ensp;用域管理账号登录主域控，使用工具mimikatz.exe执行命令抓取hash（在域控中执行）：

```cmd
#mimikatz.exe “privilege::debug” “sekurlsa::logonpasswords” exit>log.txt
```

![mimiakatz hash](https://img-blog.csdnimg.cn/20200313125259284.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2MzM0NDY0,size_16,color_FFFFFF,t_70)

## 3.在Client端查看SID号
&ensp;&ensp;&ensp;&ensp;在Client端查看SID号（在Cient端执行），复制并保存，也可以根据上一步mimikatz导出的文件中找到。

![SID](https://img-blog.csdnimg.cn/20200313125458331.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2MzM0NDY0,size_16,color_FFFFFF,t_70)

## 4.创建票据

&ensp;&ensp;&ensp;&ensp;将在域控上抓取的hash也就是NTML值的复制到Client端，打开mimikatz.exe工具(在Cient端执行)，创建票据

```cmd
#kerberos::golden /domain:<域名> /sid:<域 SID> /target:<目标服务器主机名> /service:<服务类型> /rc4:<NTLM Hash> /user:<用户名> /ptt
```

![09](https://img-blog.csdnimg.cn/2020031312551040.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2MzM0NDY0,size_16,color_FFFFFF,t_70)

```txt
kerberos::golden：使用minikatz中票据的功能
/domain：指定域名
/sid：Client端查询的sid号，在域控中查询也可以，都是一样的
/target：主域控中的计算机全名
/rc4：在域控中抓取的hash(NTLM)
/service：需要伪造的服务（cifs只是其中的一种服务，可伪造的服务很多）
/user：需要伪造的用户名（可自定义）
/ppt：伪造了以后直接写入到内存中
```

## 5.查看票据

&ensp;&ensp;&ensp;&ensp;执行了后如果提示successfully表示伪造的白银票据成功写入到内存中。可以通过kerberos::list来查看

 ![查看票据](https://img-blog.csdnimg.cn/20200313125523304.png)

## 6.白银票据攻击

&ensp;&ensp;&ensp;&ensp;打开Client的cmd来执行共享文件查看，可以看到是可以成功查看域控c盘下的文件，并且此时权限也是最高的权限。

 ![白银票据攻击](https://img-blog.csdnimg.cn/20200313125535342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2MzM0NDY0,size_16,color_FFFFFF,t_70)

# 0x05 **白银票据可伪造服务类型**

|服务注释| 服务名 |
|--|--|
| WMI | HOST、RPCSS |
|Powershell Remoteing|HOST、HTTP|
| WinRM | HOST、HTTP |
|Scheduled Tasks|HOST|
| WMI | HOST、RPCSS |
|LDAP 、DCSync|LDAP|
| Windows File Share (CIFS) | CIFS |
| Windows Remote Server Administration Tools | RPCSS、LDAP、CIFS |

# 0x06 检测白银票据利用行为

```txt
银票活动可能具有以下问题之一：
帐户域字段应为DOMAIN时为空白
帐户域字段应为DOMAIN时为DOMAIN FQDN。
事件ID：4624（帐户登录）
帐户域是FQDN，应为短域名
帐户域：LAB.ADSECURITY.ORG[ADSECLAB]
事件ID：4634（帐户注销）
帐户域为空，应为短域名
帐户域：_______________[ADSECLAB]
事件ID：4672（管理员登录）
帐户域为空，应为短域名
帐户域：_______________[ADSECLAB]
```

**不过根据本人测试采集到的日志信息，未发现此类异常日志。可能是哪里存在问题，欢迎大佬们指点**

# 0x07 白银/黄金区别

1)访问权限不同
Golden Ticket: 伪造TGT,可以获取任何Kerberos服务权限
Silver Ticket: 伪造TGS,只能访问指定的服务

2)加密方式不同
Golden Ticket由Kerberos的Hash加密
Silver Ticket由服务账号(通常为计算机账户)Hash加密

3)认证流程不同
Golden Ticket的利用过程需要访问域控,而Silver Ticket不需要

# 0x08 参考链接

白银票据与黄金票据探究
<http://sh1yan.top/2019/06/03/Discussion-on-Silver-Bill-and-Gold-Bill/>

windows认证-白银票据、黄金票据分析及利用
<http://www.h0r2yc.cn/2019/08/17/windows认证-白银票据、黄金票据分析及利用/>

[域渗透]-Pass the Ticket之金票&银票
<http://www.test666.me/archives/264/>
