title: 从getaddinfo谈谈域名服务器
date: 2015-12-27 15:46:42
tags:
- linux
- dns
categories:
- linux

toc: ture

---

最近在看**UNIX网络编程**第十一章名字与地址转换,之前在环境编程也有看到这章提到的函数,但是似懂非懂,而且压根不知道为什么建立服务器不直接用sockt直接绑定到一个struct sockaddr,而是用getaddrinfo获取服务器的ip,然后一个一个绑定监听.看了这章之后,我就像打通了任督二脉,知识之间连在一起了.

其实gethostbyname和getaddrinfo这些函数没啥好说的,因为书本上都有,自己去好好看书,好好理解足矣.这里我要总结的是关于这两个函数之后的东西:域名解析.

# 域名服务器

-----

有点计算机常识的都知道,当我们访问一个网站的时候,需要先到域名服务器获取这个域名的IP地址,然后在向这个IP地址(通常是一个服务器的地址)获取网页.这里我先谈谈这个域名服务器,可以先看这个网址[DNS消息格式](http://www.cnblogs.com/cobbliu/archive/2013/04/02/2996333.htm ""),了解DNS域名服务器的消息格式,至少要知道以下几个:
1. A  A记录是把一个主机名映射成一个32位的IPv4地址,书本上有个例子,表示freebsd这个主机对应的IPv4地址为12.106.32.254
```
freebsd IN A      12.106.32.254
	IN AAAA   3ffe:b80:1f8d:1:a00:20ff:fea7:686b
	IN MAX    5  freebsd.unpbook.com
	IN MAX    10 mailhost.unpbook.com
```
2. AAAA  称为"四A",AAAA记录是把一个主机名映射称一个128位的IPv6地址.
3. PTR   称为"指针记录",表示把IP地址映射成主机名对于IPv4地址,32位地址的4个字节先反转顺序,每个字节都转换成各自的十进制ASCLL值(0~255)后,再天使in-addr.arpa.结果字符串用于PTR查询.对于IPv6地址,128位地址中的32个四位组反转顺序,每个四位组都被转换成相应的十六禁止的ASCII值(0~9,a~f)后再添上ip6.arpa.上例主机freebsd的两个PTR为:254.32.106.12.in-addr.arpa和b.6.8.6.7.a.e.f.f.f.0.2.2.0.0.a.0.1.0.0.0.d.8.f.1.0.8.b.0.e.f.f.3.ip6.arpa
4. MX  MX记录把一个主机指定作为给定主机的"邮件交换器",例如，当Internet上的某用户要发一封信给 user@mydomain.com 时，该用户的邮件系统通过DNS查找mydomain.com这个域名的MX记录，如果MX记录存在， 用户计算机就将邮件发送到MX记录所指定的邮件服务器上。
5. CNAME  CNAME代表"canonical name"(规范名字),它的常见用法就是为常见的服务(如ftp和www)指派CNAME记录,其实就是别名.例如一台linux的主机有以下两个CNAME记录:
```
ftp  IN  CNAME  linux.unpbook.com
www  IN  CNAME  linux.unpbook.com
```
那么当查询主机名ftp或者www的ip地址,会直接获取linux.unpbook.com的A记录

# dnsmasq守护进程

----

书本提到gethostbyname,gethostbyaddr和getaddrinfo三个函数无非就是在内核执行了一次域名解析过程,解析过程如下:
![客户,解析器和域名服务器的关系](http://7xjnip.com1.z0.glb.clouddn.com/ldw-%E9%80%89%E5%8C%BA_051.png "")
图中解析器代码就是指上述提到的三个函数内核执行的代码.当调用gethostbyname函数时,在内执行这段代码时,会先到配置文件/etc/resolv.conf文件查到本地的dns服务器ip地址,然后向本地dns服务器建立udp请求信息,获取信息.

ubuntu默认的本地dns服务器地址是127.0.1.1,也就是本地的dnsmasq守护进程,即服务.我们可以打开/etc/resolv.conf文件,内容如下:
```
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 127.0.1.1
```
127.0.1.1也是一个本地回环地址,而dnsmasq进程监听的正是这个地址以及53端口号.我们也可以配置其他计算机为本地DNS域名服务器.

可以通过ps -ef | grep dnsmasq来查看这个进程:
```
➜  ~  ps -ef | grep dnsmasq
nobody    1134   867  0 15:22 ?        00:00:00 /usr/sbin/dnsmasq --no-resolv --keep-in-foreground --no-hosts --bind-interfaces --pid-file=/run/sendsigs.omit.d/network-manager.dnsmasq.pid --listen-address=127.0.1.1 --conf-file=/var/run/NetworkManager/dnsmasq.conf --cache-size=0 --proxy-dnssec --enable-dbus=org.freedesktop.NetworkManager.dnsmasq --conf-dir=/etc/NetworkManager/dnsmasq.d
charles  28729  2728  0 17:32 pts/12   00:00:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn dnsmasq
```

所以gethostbyname,gethostbyaddr和getaddrinfo这三个函数执行原理就是在内核通过udp向dnsmasq查询地址,如果dnsmasq没有相关的ip地址,那么dnsmasq会向其他域名服务器查询,最后返回到本地域名服务器缓存中.

gethostbyname主要是获取A记录,所以只能返回IPv4地址
gethostbyaddr主要是在in_addr.arpa域中向一个域名服务器查询PTR记录.
getaddrinfo函数很强大,最新的程序几乎都用这个函数,因为
1. 他既可以查询IPv4地址,也可以查询IPv6地址.
2. 当在建立服务器时,用这个函数可以获取服务器的所有ip地址(包括tcp和udp以及raw),监听所有接口.很多服务器是多宿的,即有很多网络接口,支持多个ip地址.
3. 当客户端要连接这个服务器时,可以调用获取这个服务器的所有tcp地址,然后一个一个connect,当在某个ip地址成功之后,就停止connect.
