### 后台运行命令

`nohup command &`

- nohup 忽略内部的挂断信号,不挂断的运行
- & 指后台运行

终止nohup &
```shell script
ps -aux | grep chat.js
ps -aux|grep chat.js| grep -v grep
lsof -i:8090
netstat -ap|grep 8090
kill -9  进程号
```

### 修改用户组,用户,权限
```shell script
chown [-R] 账号名称 文件或目录

chown [-R] 账号名称:用户组名称 文件或目录

chgrp [-R] 用户组名称 dirname/filename ...

```

### curl和wget命令

> curl部分转自阮一峰的网络日志 curl网站开发指南

curl这个命令本身，就是一个无比有用的网站开发工具，请看我整理的它的用法。

curl是一种命令行工具，作用是发出网络请求，然后得到和提取数据，显示在"标准输出"（stdout）上面。

它支持多种协议，下面举例讲解如何将它用于网站开发。

#### 1.查看网页源码

直接在curl命令后加上网址，就可以看到网页源码。我们以网址www.sina.com为例（选择该网址，主要因为它的网页代码较短）：
```shell script
curl www.sina.com

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://www.sina.com.cn/">here</a>.</p>
</body></html>
```
如果要把这个网页保存下来，可以使用`-o`参数，这就相当于使用wget命令了。
```shell script
curl -o [文件名] www.sina.com
```
#### 2.自动跳转

有的网址是自动跳转的。使用`-L`参数，curl就会跳转到新的网址。

　　$ curl -L www.sina.com

键入上面的命令，结果就自动跳转为www.sina.com.cn。

#### 3.显示头信息
`-i`参数可以显示http response的头信息，连同网页代码一起。

```SHELL SCRIPT
curl -i  https://ip.cn
HTTP/2 200 
date: Sun, 24 Nov 2019 15:07:45 GMT
content-type: application/json; charset=UTF-8
set-cookie: __cfduid=d35f1825cc024b23b7cd50b253f4f5d131574608065; expires=Tue, 24-Dec-19 15:07:45 GMT; path=/; domain=.ip.cn; HttpOnly
cf-cache-status: DYNAMIC
expect-ct: max-age=604800, report-uri="https://report-uri.cloudflare.com/cdn-cgi/beacon/expect-ct"
server: cloudflare
cf-ray: 53ac4b59ae8eeb8d-LAX

{"ip": "114.55.243.225", "country": "浙江省杭州市", "city": "阿里云"}

```
`-I`参数则是只显示http response的头信息。

#### 显示通信过程

`-v`参数可以显示一次http通信的整个过程，包括端口连接和http request头信息。
```shell script
 curl -v  https://ip.cn
* Rebuilt URL to: https://ip.cn/
*   Trying 104.16.25.99...
* TCP_NODELAY set
* Connected to ip.cn (104.16.25.99) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS Unknown, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Unknown (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Client hello (1):
* TLSv1.3 (OUT), TLS Unknown, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: C=US; ST=CA; L=San Francisco; O=CloudFlare, Inc.; CN=sni.cloudflaressl.com
*  start date: Feb 11 00:00:00 2019 GMT
*  expire date: Feb 11 12:00:00 2020 GMT
*  subjectAltName: host "ip.cn" matched certs "ip.cn"
*  issuer: C=US; ST=CA; L=San Francisco; O=CloudFlare, Inc.; CN=CloudFlare Inc ECC CA-2
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* TLSv1.3 (OUT), TLS Unknown, Unknown (23):
* TLSv1.3 (OUT), TLS Unknown, Unknown (23):
* TLSv1.3 (OUT), TLS Unknown, Unknown (23):
* Using Stream ID: 1 (easy handle 0x55594a44a580)
* TLSv1.3 (OUT), TLS Unknown, Unknown (23):
> GET / HTTP/2
> Host: ip.cn
> User-Agent: curl/7.58.0
> Accept: */*
> 
* TLSv1.3 (IN), TLS Unknown, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS Unknown, Unknown (23):
* Connection state changed (MAX_CONCURRENT_STREAMS updated)!
* TLSv1.3 (OUT), TLS Unknown, Unknown (23):
* TLSv1.3 (IN), TLS Unknown, Unknown (23):
* TLSv1.3 (IN), TLS Unknown, Unknown (23):
< HTTP/2 200 
< date: Sun, 24 Nov 2019 15:05:36 GMT
< content-type: application/json; charset=UTF-8
< set-cookie: __cfduid=d5fc5b00fb926d0faeaffdb2a9ebfb0011574607936; expires=Tue, 24-Dec-19 15:05:36 GMT; path=/; domain=.ip.cn; HttpOnly
< cf-cache-status: DYNAMIC
< expect-ct: max-age=604800, report-uri="https://report-uri.cloudflare.com/cdn-cgi/beacon/expect-ct"
< server: cloudflare
< cf-ray: 53ac48356b47d386-LAX
< 
{"ip": "114.55.243.225", "country": "浙江省杭州市", "city": "阿里云"}
* TLSv1.3 (IN), TLS Unknown, Unknown (23):
* Connection #0 to host ip.cn left intact

```

如果你觉得上面的信息还不够，那么下面的命令可以查看更详细的通信过程。

` curl --trace output.txt www.sina.com`

或者

` curl --trace-ascii output.txt www.sina.com`

运行后，请打开output.txt文件查看。

#### 5.发送表单信息

发送表单信息有GET和POST两种方法。GET方法相对简单，只要把数据附在网址后面就行。

`$ curl example.com/form.cgi?data=xxx`

POST方法必须把数据和网址分开，curl就要用到--data参数。

`$ curl -X POST --data "data=xxx" example.com/form.cgi`

如果你的数据没有经过表单编码，还可以让curl为你编码，参数是`--data-urlencode`。

`$ curl -X POST --data-urlencode "date=April 1" example.com/form.cgi`

#### 6.HTTP动词

curl默认的HTTP动词是GET，使用`-X`参数可以支持其他动词。
```shell script
$ curl -X POST www.example.com
$ curl -X DELETE www.example.com
````
#### 7.文件上传

假定文件上传的表单是下面这样：
```ecmascript 6
<form method="POST" enctype="multipart/form-data" action="upload.cgi">
　　<input type="file" name="upload">
　　<input type="submit" name="press" value="OK">
</form>
```
你可以用curl这样上传文件：

`$ curl --form upload=@localfilename --form press=OK [URL]`

#### 8.Referer字段

有时你需要在http request头信息中，提供一个referer字段，表示你是从哪里跳转过来的。

`$ curl --referer http://www.example.com http://www.example.com`

#### 9.User Agent字段

这个字段是用来表示客户端的设备信息。服务器有时会根据这个字段，针对不同设备，返回不同格式的网页，比如手机版和桌面版。

curl可以这样模拟：

`$ curl --user-agent "[User Agent]" [URL]`

#### 10.cookie

使用`--cookie`参数，可以让curl发送cookie。

`$ curl --cookie "name=xxx" www.example.com`

至于具体的cookie的值，可以从http response头信息的`Set-Cookie`字段中得到。

`-c cookie-file`可以保存服务器返回的cookie到文件，`-b cookie-file`可以使用这个文件作为cookie信息，进行后续的请求。
```shell script
$ curl -c cookies http://example.com
$ curl -b cookies http://example.com
````
#### 11.增加头信息

有时需要在http request之中，自行增加一个头信息。`--header`参数就可以起到这个作用。

`$ curl --header "Content-Type:application/json" http://example.com`

#### 12.HTTP认证

有些网域需要HTTP认证，这时curl需要用到`--user`参数。

`$ curl --user name:password example.com`

### scp命令
```
一、scp是什么？
scp是secure copy的简写，用于在Linux下进行远程拷贝文件的命令，和它类似的命令有cp，不过cp只是在本机进行拷贝不能跨服务器，而且scp传输是加密的。可能会稍微影响一下速度。

二、scp有什么用？
1、我们需要获得远程服务器上的某个文件，远程服务器既没有配置ftp服务器，没有开启web服务器，也没有做共享，无法通过常规途径获得文件时，只需要通过scp命令便可轻松的达到目的。

2、我们需要将本机上的文件上传到远程服务器上，远程服务器没有开启ftp服务器或共享，无法通过常规途径上传是，只需要通过scp命令便可以轻松的达到目的。

三、scp使用方法
1、获取远程服务器上的文件
scp -P 2222 root@www.vpser.net:/root/lnmp0.4.tar.gz /home/lnmp0.4.tar.gz

上端口大写P 为参数，2222 表示更改SSH端口后的端口，如果没有更改SSH端口可以不用添加该参数。 root@www.vpser.net 表示使用root用户登录远程服务器www.vpser.net，:/root/lnmp0.4.tar.gz 表示远程服务器上的文件，最后面的/home/lnmp0.4.tar.gz表示保存在本地上的路径和文件名。还可能会用到p参数保持目录文件的权限访问时间等。

2、获取远程服务器上的目录
scp -P 2222 -r root@www.vpser.net:/root/lnmp0.4/ /home/lnmp0.4/

上端口大写P 为参数，2222 表示更改SSH端口后的端口，如果没有更改SSH端口可以不用添加该参数。-r 参数表示递归复制（即复制该目录下面的文件和目录）；root@www.vpser.net 表示使用root用户登录远程服务器www.vpser.net，:/root/lnmp0.4/ 表示远程服务器上的目录，最后面的/home/lnmp0.4/表示保存在本地上的路径。

3、将本地文件上传到服务器上
scp -P 2222 /home/lnmp0.4.tar.gz root@www.vpser.net:/root/lnmp0.4.tar.gz

上端口大写P 为参数，2222 表示更改SSH端口后的端口，如果没有更改SSH端口可以不用添加该参数。 /home/lnmp0.4.tar.gz表示本地上准备上传文件的路径和文件名。root@www.vpser.net 表示使用root用户登录远程服务器www.vpser.net，:/root/lnmp0.4.tar.gz 表示保存在远程服务器上目录和文件名。

4、将本地目录上传到服务器上
scp -P 2222 -r /home/lnmp0.4/ root@www.vpser.net:/root/lnmp0.4/

上 端口大写P 为参数，2222 表示更改SSH端口后的端口，如果没有更改SSH端口可以不用添加该参数。-r 参数表示递归复制（即复制该目录下面的文件和目录）；/home/lnmp0.4/表示准备要上传的目录，root@www.vpser.net 表示使用root用户登录远程服务器www.vpser.net，:/root/lnmp0.4/ 表示保存在远程服务器上的目录位置。

5、可能有用的几个参数 :
-v 和大多数 linux 命令中的 -v 意思一样 , 用来显示进度 . 可以用来查看连接 , 认证 , 或是配置错误 .

-C 使能压缩选项 .

-4 强行使用 IPV4 地址 .

-6 强行使用 IPV6 地址 .

```

### 输入输出定向 & EOF



在平时的运维工作中，我们经常会碰到这样一个场景：
执行脚本的时候，需要往一个文件里自动输入N行内容。如果是少数的几行内容，还可以用echo追加方式，但如果是很多行，那么单纯用echo追加的方式就显得愚蠢之极了！
这个时候，就可以使用EOF结合cat命令进行行内容的追加了。

下面就对EOF的用法进行梳理：
EOF是END Of File的缩写,表示自定义终止符.既然自定义,那么EOF就不是固定的,可以随意设置别名,在linux按ctrl-d就代表EOF.
EOF一般会配合cat能够多行文本输出.
其用法如下:
<<EOF        //开始
....
EOF            //结束

还可以自定义，比如自定义：
<<BBB        //开始
....
BBB              //结束

通过cat配合重定向能够生成文件并追加操作,在它之前先熟悉几个特殊符号:
< :输入重定向
> :输出重定向
>> :输出重定向,进行追加,不会覆盖之前内容
<< :标准输入来自命令行的一对分隔号的中间内容.

下面通过具体实例来感受下EOF用法的妙处：
1）向文件test.sh里输入内容。
```shell script
[root@slave-server opt]# cat << EOF >test.sh 
> 123123123
> 3452354345
> asdfasdfs
> EOF
[root@slave-server opt]# cat test.sh 
123123123
3452354345
asdfasdfs
```

追加内容
```shell script
[root@slave-server opt]# cat << EOF >>test.sh 
> 7777
> 8888
> EOF
[root@slave-server opt]# cat test.sh 
123123123
3452354345
asdfasdfs
7777
8888
```
```shell script

覆盖
[root@slave-server opt]# cat << EOF >test.sh
> 55555
> EOF
[root@slave-server opt]# cat test.sh 
55555

```


2）自定义EOF，比如自定义为wang
```shell script
[root@slave-server opt]# cat << wang > haha.txt
> ggggggg
> 4444444
> 6666666
> wang
[root@slave-server opt]# cat haha.txt 
ggggggg
4444444
6666666       
```

3）可以编写脚本，向一个文件输入多行内容
```shell script
[root@slave-server opt]# touch /usr/local/mysql/my.cnf               //文件不提前创建也行，如果不存在，EOF命令中也会自动创建
[root@slave-server opt]# vim test.sh
#!/bin/bash

cat > /usr/local/mysql/my.cnf << EOF                                      //或者cat << EOF > /usr/local/mysql/my.cnf
[client]
port = 3306
socket = /usr/local/mysql/var/mysql.sock
EOF

[root@slave-server opt]# sh test.sh           //执行上面脚本
[root@slave-server opt]# cat /usr/local/mysql/my.cnf    //检查脚本中的EOF是否写入成功
[client]
port = 3306
socket = /usr/local/mysql/var/mysql.sock
```

