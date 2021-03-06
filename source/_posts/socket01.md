title: linux网络学习前五章读书笔记
date: 2015-11-28 15:35:35
tags:
- Socket
categories:
- Socket
toc: true

---

linux系统最重要的两块知识:进程(线程)间的通信与同步和网络编程.之前暑期已看了一遍**进程间的通信**,从今天开始,打算好好看下**网络编程**.

因为已经看过**unix环境高级编程**和redis的源码,所以对网络编程的基础api大概都懂,但是对于原理性的东西,啥都不知道.所以打算看**网络编程**这本书,以及后期看libevent和memchached源码实现,巩固下所学知识.我看了前五章,发现这本书的知识正是我需要的,所以对一些知识点,做个总结.

# TCP连接的建立

-----

两台计算机网络的建立,一定经过三次握手过程,先上示意图,来自**网络编程卷1**
![TCP连接建立和断开](http://7xjnip.com1.z0.glb.clouddn.com/ldw-选区_041.png "")
首先要说明的是,客户端通过建立socket并调用connect连接服务器为主动打开,而服务器通过调用socket,bind,listen准备客户连接为被动过程.
1. 首先服务器端通过调用socket,bind,listen监听服务器端的套接字,然后调用accept并阻塞在accept,等待客户连接.
2. 客户端先调用socket创建套接字,然后阻塞在connect调用.这时内核将自动执行三次握手过程.当客户端向服务器端发送一个syn时，这时客户端处于SYN_SENT状态；当服务器收到客户发来的syn时，处于SYN_RECV状态，并返回ack和syn；
3. 当客户端收到服务器端发来的syn和ack时,返回一个ack并从connect返回,表明已连上服务器.此时客户端为ESTABLISHED状态；
3. 服务器端收到客户端的ack之后,从accept返回一个服务器套接字,这个套接字与客户端已建立连接,此时服务器处于ESTABLISHED状态，接着客户端一般情况下就阻塞在read调用.

# TCP连接终止

---

四次挥手过程示意图如上．
对于连接终止过程，可以由客户端或服务器端任何一端先发起，先发起断开的一端为主动关闭，则另一端为被动关闭．假设由先客户发起
1. 客户端调用close关闭套接字时，内核会向服务器端发生一个fin分节，此时客户端处于FIN_WAIT_1状态；
2. 服务器端收到客户端的fin分节之后，服务端反馈一个ack，并从read返回0字节，此时服务器处于CLOSE_WAIT；
3. 当客户端收到服务器端返回的ack时，客户端处于FIN_WAIT_2状态；
4. 当服务器端也调用close时，向客户发送一个fin，客户接收到fin之后，处于TIME_WAIT状态，等待2MSL，客户端就关闭了
5. 服务器端收到收到客户端的ack之后，立即处于CLOSED状态，即关闭连接．

服务器端在CLOSE_WAIT和LAST_ACK状态之间还是可以发送数据的,但是客户端都已调用close函数,接下来也不会对这个数据进行处理,所以没啥意义.

为什么主动关闭的那端会有TIME_WAIT状态了?
1. 可靠地实现TCP全双工连接的终止;
2. 允许老的重复分节在网络中消逝.

对于第一点,我们可以假设最后那个ack丢失的情况.当最后ack丢失之后,服务器端因为没有收到ack,所以必须重发fin,这就导致客户端必须重发ack.从客户端发第一个ack到重发ack刚好是两倍的包存活最长时间MSL.(一个是第一个ack存活时间,一个是重传的fin包)

对于第二点,有足够的时间让这个连接不会跟后面的连接混在一起（你要知道，有些自做主张的路由器会缓存IP数据包，如果连接被重用了，那么这些延迟收到的包就有可能会跟新连接混在一起）。

# TCP基础知识

-----

1. MSS:MSS 最大分节大小.在数据链路层,有MTU这个概念,解释为某个信道最大的通信量.所以如果IP层,数据量太大的话,必须将ip包进行分片传输,然后在接收端进行包重组,这样以来,会影响到效率问题.所以linux内核就在TCP阶段,将传给IP层的数据限制在一定大小,防止分片.而TCP这个数据量的大小就是MSS,即TCP层最大传输数据段大小.这个大小一般等于MTU-20(IP包头)-20(TCP包头),这样就可以防止IP层分片了.

+ 当在连接建立开始阶段,双方会互相发送各自MSS的大小,最后通过协商,选择二者较小的数值作为双方的MSS

+ 当前以太网的MTU=1500,所以ipv4的MSS=1460,ipv6的MSS=1440(ipv6的首部为40字节).而在ipv4和ipv6都定义了最小重组缓冲区的大小,即ipv4和ipv6任何实现都必须保证支持的最小数据报的大小.ipv4最小重组缓冲区为576,ipv6为1500,这时的MSS分别为536和1440.

2. ipv4首部有个"不分片位(DF)",如果被设置了,则不管是主机还是路由器都不准对这个数据报分片,那么当传到数据链路层时,如果大于MTU,则会产生一个ICMP"目的地不可达,需要分片但是DF位已设置"的错误消息.

3. TCP分片还有一个窗口选项,用于向对端通知从ack序列号+1开始,还可以接收多少数据量.

4. TCP为了拥塞避免,设置了一个拥塞窗口,拥塞窗口的大小为此时可以发送的数据量,以tcp报文为单位,则可以用MSS的个数来表示可以发送的数据量.TCP用慢启动和快重传来避免网络拥塞.
5. 判断系统大小端,可以用如下联合体来判断:
```
union {
	short s;
	char c[sizeof(short)];
}un
```
# socket相关函数解释

---

## connect函数

当客户端调用socket获取一个套接字时,即可调用connect连接服务器.在调用connect之前,其实是可以调用bind绑定本地ip地址和端口,但是没必要,因为内核会自己选择一个ip地址和临时端口.

当调用connect时,用户进程则会阻塞,内核则会触发三次握手过程,并且只有在建立成功或者出错时才返回,出错时,有以下情况:
1. 若TCP客户没有收到SYN分节的响应,则返回ETIMEDOUT错误.例如此时服务端还没开启时.
2. 若对客户的SYN的响应是RST(表示复位),则表明该服务器主机在我们指定的端口没有进程在与之连接,返回ECONNREFUCED错误.

RST是TCP发生错误时发送的一种TCP分节.产生RST的三个条件是:目的地的目的端口SYN到达,但却没有服务器监听这个端口;TCP想取消一个已有连接;TCP接收到一个根本不存在的连接上的分节.
3.若客户发出的SYN在中间某个路由器上引发了一个"destination unreachable"的ICMP错误,为软错误,则ICMP错误会作为EHOSTUNREACH或ENETUNREACH错误返回给进程.

对于错误原因的研究非常重要,因为在服务器出现问题时,可以分析错误的根源,便于快速定位到错误.

## bind函数

bind函数用于将ip地址和端口绑定到一个套接字上,可以指定一个ip地址,或者指定一个端口,或者两个都指定,或者两个都不指定.

对于客户端而言,一般情况下不绑定ip地址和端口,调用connnect时,由内核选择.
对于服务端而言,我们可以设置ip地址和端口,有如下组合:
![bind函数ip地址和端口设置组合](http://7xjnip.com1.z0.glb.clouddn.com/ldw-选区_042.png "")

ip地址的通配地址为INADDR_ANY,其值一般全为0,它告诉内核自己选择ip地址,一般就是回环地址127.0.0.1和本机局域网地址.

实际开发中,服务器端口肯定是固定的,用于客户端连接,ip地址可以用通配符的格式,这样即可以本机连接,也可以其他机子连接.

## listen函数

通过看这本书,我对listen函数从只会调用,到理解内核原理跨越.当socket创建套接字时,它默认被设置为主动套接字,也就是说它是一个调用connect发起主动连接的客户套接字.listen函数将这个套接字转换为一个被动套接字,指示内核应该接受指向该套接字的连接请求.

listen函数的第二个参数是规定了内核应该为相应套接字排队的最大连接个数.内核为每个监听套接字维护两个队列:
1. 未完成连接队列 每个SYN分节对应其中一项:已由某个客户发出并达到服务器,而服务器正在等待完成相应的TCP三路握手过程,这些套接字处于SYN_RCVD状态.
2. 已完成连接队列 每个已完成TCP三路握手过程的客户对应其中一项,这些套接字处于ESTABLISHED状态.

下图描绘了监听套接字的两个队列:
![TCP为套接字维护的两个队列](http://7xjnip.com1.z0.glb.clouddn.com/ldw-选区_043.png "")

当进程调用accept时,已完成队列的队头项将返回给进程;如果该队列为空,则进程将阻塞在accept函数中.

listen函数的第二参数backlog参数为这两个队列的总和的最大值或者增加一个模糊因子:乘以1.5,例如通常backlog设为5,所以最大套接字排队为1.5*5=8个.

在正常情况下(没有丢失,没有重传),未完成队列的每一项在其中的存活时间为RTT,如下图所示:
![分节RTT存活时间](http://7xjnip.com1.z0.glb.clouddn.com/ldw-选区_044.png "") 


当一个客户SYN达到时,若队列是满的,TCP就忽略该分节,客户会重新发送SYN分节.因为这只是暂时的,很快队列就会腾出空位.如果一开始就返回一个RST错误,那么客户不知道这个RST是表示"该端口没有服务器监听",还是意味着"该端口有服务器在监听,不过他的队列满了".

在三次握手完成后,但在调用accept之前达到的数据应由服务器TCP排队,最大数据量为相应已连接套接字的接收缓冲区大小.

## accept函数

accept函数由TCP服务器调用,用于从以完成连接队列对头返回下一个已完成的连接,如果队列为空,则阻塞.

accept后两个参数分别为客户端的地址以及该地址的大小,如果不感兴趣,可以同时设置为NULL.

# 并发服务器

---

现在并发程序的设计框架无非就是单进程(IO多路复用),多进程和多线程,网络编程这本书一开始先介绍了多进程编程.

并发多进程的实现主要是通过fork来实现的.

在多进程编程时,还需要考虑一个问题,即子进程结束时,避免成为僵尸进程.避免僵尸进程有三种方法:
1. 调用wait或者waitpid函数来等待子进程结束,并回收资源
2. 调用两次fork函数,即由父进程派生子进程,子进程派生孙进程,接着子进程结束.这样能避免僵尸进程主要原因是当子进程派生孙进程而结束时,子进程可以由父进程回收资源,然后孙进程由与子进程结束,就被过继给init进程,init进程会在孙进程结束时回收资源.
3. 设置SIG_CHID处理函数为SIG_IGN
但是为了可移植性,在多进程程序中,避免僵尸进程的主要方法是在父进程捕获SIG_CHID信号,然后在处理函数中调用wait函数.

在调用wait函数时,又出现了问题,这里就不介绍wait和waitpid函数的使用方法,主要讲下书本为什么调用waitpid而不是wait.

在处理函数中,如果调用wait函数时为
```
pid=wait(&stat);
```
如果为waitpid函数时为:
```
while( (pid=waitpid(-1,&stat,WNOHANG)) > 0)
    printf("child %d terminated\n",pid);
```
书中例子,在客户端产生了5个套接字连接服务器,这导致服务器产生5个进程来处理这5个客户端.当客户终止时,这时几乎同时产生5个FIN发到服务器,服务器对于第一个结束的子进程,此时服务器执行于SIG_CHID处理函数中,后面达到的SIG_CHID信号则排队,因为SIG_CHID是不可靠信号,是不会一个一个执行的,最终只会处理调用两次信号处理函数,说明还有三个僵尸进程为处理.

如果调用的是wait函数,就是上述描述的情况,会产生三个僵尸进程;

如果是调用waitpid函数,-1表示等待第一个结束的进程,WNOHANG告诉内核在没有已终止子进程时不要阻塞,所以对于同时产生的5个SIG_CHID信号,则可以调用一次信号处理函数,在while循环中把5个子进程资源都回收了.如果把while循环调用的是wait,也不行,因为没办法防止wait在正运行的子进程尚有未终止时阻塞.

如果信号是一个一个到达,也就是一个信号处理函数结束,再产生第二个SIG_CHID信号,上述两个都可以,但是对于同时产生信号而言,只能在while循环中调用waitpid函数.

:wq



未完待续...





























