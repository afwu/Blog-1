# CobaltStrike去除流量特征

首先说明，本文非原创。转载！！！最近有计划的学习相关知识，自己也是按照这篇文章进行浮现的。以下为文章原文：

​普通CS没有做流量混淆会被防火墙拦住流量，所以偶尔会看到CS上线了机器但是进行任何操作都没有反应。这里尝试一下做流量混淆。参考网上的文章，大部分是两种方法，一种更改teamserver 里面与CS流量相关的内容，一种是利用Keytool工具生成新的store证书。而我们需要做的修改大概为3个地方：

1. 修改默认端口
2. 去除store证书特征
3. 修改profile

## 0x00 关闭后台运行的CS

```yml
ps -aux
找到cs相关的进程
kill -9 pid
```

## 0x01 修改默认端口

编辑teamserver文件，更改server port部分 50433

```bash
vim teamserver
```

ps:这里我直接在本地用记事本打开的，如果是在linux系统可以使用vim\gedit等方式编辑

![修改默认端口](https://img2020.cnblogs.com/blog/1835657/202103/1835657-20210311153132879-1272063410.png)

## 0x02 去除store证书特征

查看证书，默认密码123456

```bash
keytool -list -v -keystore cobaltstrike.store
```

![修改store特征1](https://img2020.cnblogs.com/blog/1835657/202103/1835657-20210311153131925-1937699290.png)

而Keytool是一个Java的证书管理工具，下面用Keytool生成一个store证书。

```bash
keytool -h                                                                                                                                                                                                       
Illegal option:  -h                                                                                                                                                                                              
ey and Certificate Management Tool                                                                                                                                                                              


Commands:                                                                                                                                                                                                        

-certreq            Generates a certificate request                                                                                                                                                              

-changealias        Changes an entry's alias                                                                                                                                                                     

-delete             Deletes an entry                                                                                                                                                                             

-exportcert         Exports certificate                                                                                                                                                                          

-genkeypair         Generates a key pair                                                                                                                                                                         

-genseckey          Generates a secret key                                                                                                                                                                       

-gencert            Generates certificate from a certificate request                                                                                                                                             

-importcert         Imports a certificate or a certificate chain                                                                                                                                                 

-importpass         Imports a password                                                                                                                                                                           

-importkeystore     Imports one or all entries from another keystore                                                                                                                                             

-keypasswd          Changes the key password of an entry                                                                                                                                                         

-list               Lists entries in a keystore                                                                                                                                                                  

-printcert          Prints the content of a certificate                                                                                                                                                          

-printcertreq       Prints the content of a certificate request                                                                                                                                                  

-printcrl           Prints the content of a CRL file                                                                                                                                                             

-storepasswd        Changes the store password of a keystore           
```

使用以下命令生成一个新的store证书，-alias 和 -dname 可以自由发挥，也可以用其他的来混淆流量。

```bash
keytool -keystore CobaltStrikepro.store -storepass 123456 -keypass 123456 -genkey -keyalg RSA -alias Microsec.com -dname "CN=Microsec e-Szigno Root CA, OU=e-Szigno CA, O=Microsec Ltd., L=Budapest, S=HU, C=HU" 
                                                                                                                                                                                                             

参数               含义    
    
`-alias`          指定别名
`-storepass`      指定更改密钥库的存储口令
`‐keypass pass`   指定更改条目的密钥口令
`-keyalg`         指定算法
```

新生成的证书看着就很nice

![修改证书特征](https://img2020.cnblogs.com/blog/1835657/202103/1835657-20210311153131074-1058910214.png)

当然也可以编辑 teamserver 文件来生成证书

ps:这里我也选择的是这种方法修改证书特征

![手动修改特征](https://img2020.cnblogs.com/blog/1835657/202103/1835657-20210311153130289-1252176254.png)

## 0x03 Malleable-C2-Profiles

因为现在很多WAF都能检测出CS的流量特征，而CS的流量由Malleable C2配置来掌控的，所以我们需要定向去配置这个Malleable-C2。

**Beacon与teamserver端c2的通信逻辑**

```yml
1.stager的beacon会先下载完整的payload执行。

2.beacon进入睡眠状态，结束睡眠状态后用 http-get方式 发送一个metadata(具体发送细节可以在malleable_profie文件里的http-get模块进行自定义),metadata内容大概是目标系统的版本，当前用户等信息给teamserver端。

3.如果存在待执行的任务，则teamserver上的c2会响应这个metadata发布命令。beacon将会收到具体会话内容与一个任务id。

4.执行完毕后beacon将回显数据与任务id用post方式发送回team server端的C2(细节可以在malleable_profile文件中的http-post部分进行自定义)，然后又会回到睡眠状态。
```

首先需要先下载profile文件

```yml
git clone https://github.com/rsmudge/Malleable-C2-Profiles.git
```

CS中集成了一个包含在Linux平台下的C2lint工具，可以检查profile代码是否有问题

```yml
chmod 777 c2lint
./c2lint ./Malleable-C2-Profiles/APT/havex.profile
```

ps：记得权限要设置对

之后改一下profile的内容就好了网上有很多例子，我这里简单改了下。

因为0.0.0.0是Cobalt Strike DNS Beacon特征，可以在profile内加一段 set dns_idle "8.8.8.8"; 之后profile内默认的能改则改。

![修改通讯特征](https://img2020.cnblogs.com/blog/1835657/202103/1835657-20210311153129645-1499408261.png)

http-get部分，包括uri和header都可以根据实战抓包进行修改。

![修改http-get部分](https://img2020.cnblogs.com/blog/1835657/202103/1835657-20210311153125065-1797803502.png)

ps:本人这里都设置的是百度的，主要通过抓包分析请求百度的信息特征，进行了修改。

参考来源：

<https://www.adminxe.com/1489.html>

<https://www.chabug.org/web/832.html>

<https://paper.seebug.org/1349/>

<https://blog.csdn.net/shuteer_xu/article/details/110508415>

<https://www.cnblogs.com/Zh1z3ven/p/14518441.html>

## 0x04 后续计划

ps:此部分不属于原文内容！！！

1、最近一年来流行web漏洞学习研究；

2、cs免杀+域前置；

3、钓鱼攻击手法学习；

4、ATT&CK技战术复现，非公开；