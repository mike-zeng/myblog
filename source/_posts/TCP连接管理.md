---
title: TCP连接管理
sidebar: [grid, toc, tags] # 放置任何你想要显示的侧边栏部件
categories: [计算机网络, 传输层]
date: 2019-12-29
tags:
- tcp
- 传输层
- 连接管理
- 计算机网络
---

##### 1. TCP三次挥手

TCP是一种面向连接的单播协议，在发送数据之前，通信双方必须彼此建立一条连接。一个tcp连接由一个四元组构成（源ip地址，源端口号，目的ip地址，目的端口号）

**三次握手流程**：

<img src="https://severinblog-1257009269.cos.ap-guangzhou.myqcloud.com/TCP%E8%BF%9E%E6%8E%A5%E7%AE%A1%E7%90%86/iamge-20200131221035.png" style="zoom:50%;" />

- 主动连接者（客户端）向服务端发送一个SYN报文(TCP头部的SYN位置位并且随机生成一个seq序列号)

- 服务端发送自己的SYN报文段作为响应，同时这个报文段中携带了对上一个报文段的ACK确认（ack确认号为上一个报文的的序列号+1）

- 客户端收到第二个报文后，会对服务端的同步报文段进行确认

***

##### 2. TCP四次握手

连接释放比连接建立更加复杂，涉及到的知识点也更多

**四次挥手过程：**

<img src="https://severinblog-1257009269.cos.ap-guangzhou.myqcloud.com/TCP%E8%BF%9E%E6%8E%A5%E7%AE%A1%E7%90%86/image-20200131221721.png" style="zoom:50%;" />

- 主动关闭者发送过一个FIN报文段给被动关闭者，并且指明当前的序列号（这个报文段好可以携带同一个ack对上一个报文进行确认，这是可选的不是必须的）

- 连接的被动关闭者序列号加1发送ACK报文段给主动关闭者，此时主动关闭者到被动关闭者这一方向的连接就关闭了，主动关闭者不能发送数据给被动关闭者但是被动关闭者发送给主动关闭者的数据依然要接收。此时处于半关闭状态

- 被动关闭者发送FIN报文段个主动关闭者

- 主动关闭者对FIN报文段进行确认，此时连接已经释放，但是主动关闭者还要等一段时间

***

##### 3. TCP状态转换

**TCP三次握手状态变化**

客户端和服务端一开始都是处于**CLOSE**关闭状态，服务端需要先监听端口，此时处于**Listen**状态，具体变化过程如下：

- 第一次握手（客户端-> 服务端）

  客户端发送完之后处于**SYN_SENT**(同步已发送)

- 第二次握手（服务端->客户端）

  服务端收到第一次握手报文后立即确认，处于**SYN_RESVD**(同步已接收)

- 第三次握手（客户端发送，服务端接收）

  客户端收到第二个报文后立即确认，发送完客户端处于**ESTABLISHED**状态（已建立连接）

  服务端收到第三个报文后处于ESTABLISHED状态（已建立连接）

**TCP四次握手状态变化**

- 第一个握手报文(主动关闭者-> 被动关闭者)

  主动关闭者发送完报文后处于**FIN-WAIT-1**状态

- 第二个握手报文(被动关闭者-> 主动关闭者)

  被动关闭者发送后处于**CLOSE_WAIT**状态

  主动关闭者接收后处于**FIN-WAIT-2**状态

- 第三个握手报文(被动关闭者-> 主动关闭者)

  被动关闭者发送后处于**LAST_ACK**状态

- 第四个握手报文(主动关闭者-> 被动关闭者)

  主动关闭者发送后处于**TIME-WAIT**状态

  被动关闭者发送后处于CLOSE状态

**TCP状态详细**

TCP连接与释放总共涉及到11种状态，上面已经涉及到10种，还有一种状态是**CLOSEING**，下面是详细的TCP状态转换图。

<img src="https://severinblog-1257009269.cos.ap-guangzhou.myqcloud.com/TCP%E8%BF%9E%E6%8E%A5%E7%AE%A1%E7%90%86/image-20200131222704.png"  />

- TIME_WAIT状态

  在第三个挥手报文到达主动关闭方时，主动关闭方进入TIME_WAIT状态。TIME_WAIT状态也称2MSL等待状态，在该状态，TCP将会等待两倍于最大段生存期，MSL的值是可以修改的，一般为30秒、1分钟或者2分钟。

  TIME_WAIT状态的意义

  - 能够让TCP主动关闭者有机会重新发送ACK以避免丢失的情况

    假设第四次挥手ACK报文丢失，那么被动关闭会重新发送FIN报文，如果TCP主动关闭在不处于TIME_WAIT状态而是直接关闭了，那么被动关闭在就收不到ACK报文，会一直重发。

  - 允许老的数据包在网络中消逝。当TCP处于等待状态，通信双方将本次连接定义为不可用，		

    因为TIME_WAIT状态持续2MSL，就可以保证当成功建立一个TCP连接的时候，来自之前连接的重复分组已经在网络中消逝。

- 静默时间概念

  假设某台主机的某个连接处于TIME_WAIT状态，然后突然主机关机并在MSL内重启，重启后主机并不知道上次的连接处于TIME_WAIT状态，于是使用关机前的ip和端口建立连接就有可能出现问题。

  为了解决这个问题，引入了静默时间的概念，静默时间是指主机崩溃或重启后TCP协议应该在创建新的连接前等待MSL，但是实际上很多实现并没有遵循这一点

- FIN_WAIT_2状态

  TCP主动关闭发送FIN报文并收到ACK后会进入FIN_WAIT_2状态，而被动关闭者进入CLOSE_WAIT状态。假如被动关闭者迟迟不发生FIN报文段，那么主动关闭者会一直处于FIN_WAIT_2状态。

  为了防止TCP进入FIN_WAIT_2这一无限等待状态，如果负责主动关闭的进程执行的是一个完全关闭操作而不是半关闭操作，那么就会设置一个计时器，如果计时器超时网络是空闲的，那么TCP连接就会转移到CLOSE状态，在linux中默认是60s。

- 同时打开和同时关闭的状态转换

  - 同时打开

    ![](https://severinblog-1257009269.cos.ap-guangzhou.myqcloud.com/TCP%E8%BF%9E%E6%8E%A5%E7%AE%A1%E7%90%86/image-20200201111707.png)

    可以认为是四次握手。

  - 同时关闭

    ![](https://severinblog-1257009269.cos.ap-guangzhou.myqcloud.com/TCP%E8%BF%9E%E6%8E%A5%E7%AE%A1%E7%90%86/image-20200201111640.png)

    出现了最后一种状态**CLOSEING**状态，此时不会进入FIN_WAIT_2状态