---
title: TCP超时重传
sidebar: [grid, toc, tags] # 放置任何你想要显示的侧边栏部件
categories: [计算机网络, 传输层]
date: 2019-12-31
tags:
- 计算机网络
- 传输层
- 超时重传
---



##### 1. 简单的超时与重传举例

##### 2. 设置重传超时

TCP超时重传的基础是怎样根据给定的RTT设置RTO，如果RTO小于RTT，会导致不必要的重传，反之如果RTO远大于RTT，会导致网络利用率下降。RTT的测量非常复杂，TCP收到数据确认信息，该信息包含携带一个字节的数据来测量传输该确认信息所需时间，每个此类的测量结果称为一个RTT样本，**TCP首先要根据一段时间的RTT样本值来建立好估计值。然后基于估计值来设置RTO**。

- **经典方法**

  最初的TCP规范采用如下公式计算得到平滑的RTT估计值

  - 先计算平滑RTT估计值（称为SRTT）

    `SRTT=αSRTT+(1-α)RTT` α称为平滑因子，推荐取值为0.8到0.9

  - 采用如下公式设置RTO

    `RTO=min(ubound,max(lbound，(SRTT)β)`
    ubound为RTO上界，Ibound为RTO下界，β为时延离散因子，取值为1.3到2.0

  对于比较稳定的RTT分布，经典方法可以取得不错的效果，但是如果RTT变化较大，则无法获取期望的结果。

- **标准方法**

  标准方法是基于平均值和方差来估计的，但是方差计算复杂，不适合快速TCP的实现，因此使用计算更加简单快速的平均偏差来代替方差。因此需要计算平均值和平均偏差。

  - 计算srtt

    `srtt=(1-g)(srtt)+g(M)`

  - 计算rttvar

    `rttvar=(1-h)(rttvar)+(h)(|M-strr|)`

  - 计算RTO

    `RTO=srtt+4(rttvar)`

- **带时间戳选项的RTT测量**            

- **Linux采用的方法**

***

##### 3. 基于计时器的重传

一旦TCP发送端得到了基于时间变化的RTT测量值，就能据此设置RTO，发送报文段时确保重传计时器设置合理。在设定计时前，需记录被计时的报文段序列号，若及时收到了该报文段的ACK，那么计时器被取消。之后发送一个新的数据包，需要设定一个新的计时器，并记录新的序列号。若在连接设定的RTO内，没有收到计时报文段的ACK，将会触发超时重传，发生超时重传会通过降低当前数据发送速率来对此进行快速响应。有两种实现方式：第一种方式是基于拥塞控制机制减小发送窗口大小；另一种方法为每当一个重传报文段被再次重传时，则增加RTO的退避因子，当一个报文段出现多次重传时，RTO的值等于RTO乘以一个系数，这个系数呈指数退避变化。

***

##### 4. 快速重传

快速重传基于接收端的反馈信息来引发重传，而非重传计时器的超时。

与超时重传相比快速重传能更加有效的修复丢包情况。

当接收端收到失序数据时，应该立即返回重复ACK，不能延时发送。

**快速重传算法可以概述为：**

TCP发送端在观测到至少`dupthresh`个重复ACK后，即重传可能丢失的数据分组，而不必等到重传计时器超时。快速重传算法也可以触发拥塞控制。

***

##### 5. 带选择确认的重传 

接收端收到失序的报文段后，不使用选择确认选项会返回一个ACK对按顺序收到的最后一个报文段进行确认，失序的报文段会丢失，发送端需要重传所有失序的报文段，因此会出现重复发送多个报文段的现象，开启选择确认选项后接收放可以在ACK中告诉发送端失序数据的哪些部分需要填补，这样就避免了发送重复的报文。

每个SACK信息包含32位的序列号，代表接收端存储的失序数据的起始至最后一个序列号。每个ACK可以包含三个SACK信息，即可以向发送端报告三个空缺。

**SACK接收端行为**

 TCP连接建立期间如果运行SACK选项，接收端缓存中存在失序数据，就可以生成SACK，并且SACK信息安卓接受的先后依次排列。                                                                                                                                                                                  

**SACK发送端行为**

发送端提供SACK功能并合理地利用接收到的SACK块来进行丢失重传，该过程称为选择性重传。发送方选择重传不会清楚缓存中的书库。只有收到普通的ACK时才可清除。选择重传不会启动超时重传计时器，即使接收端仍有失序数据，那么接收端还会发送SACK信息。

***

##### 6. 伪超时与重传

某些情况下，即使没有出现数据丢失也可能引发重传。这种不必要的重传称为伪重传，其主要造成的原因是伪超时。

- 重复SACK（DSACK）

  SACK只能解决报文段失序时告知发送端，但是不能够解决接收端收到重复数据的问题。DSACK可以在第一个SACK块中告知接收端收到的重复报文段序列号，DSACK的主要目的是判断何时的重传是不必要的。DSACK接收端的的变化在于运行包含序列号小于累积ACK字段的SACK块，DSACK只包含在单个ACK中，该ACK称为DSACK，与通常的SACK'信息不同，DSACK信息不会在多个SACK中重复，因此DSACK较普通的SACK鲁棒性低，

- Eifel检测算法

- 前移RTO恢复

- Eifel响应算法

***

##### 7. 包失序与包重复

- 失序

  在IP网络中出现失序的原因在于IP层不能保证传输是有序进行的，这样做的好处是IP可以选择不同的传输链路来传输数据。失序问题可能发生在正向或反向链路（ACK报文段失序）中。

  如果失序发生在反方向，可能会使得TCP发送端窗口快速前移，接着又可能收到一些显然重复而应被丢弃的ACK，出于TCP的拥塞控制行为，会导致发送端出现不必要的流量突发（每收到一个ACK，拥塞窗口会增加，由于窗口前移了，所有窗口内可以发送的数据突然变多）

  如果失序发生在正方向，TCP可能无法正确识别丢包和失序，导致伪重传，为了解决这个问题，快速重传仅在达到重复阈值后才会被触发。互联网中严重的失序不常见，所有重复阈值设置的相对较小，默认为3。

- 重复

  包的重复会使得接收端生成一系列重复的ACK，从而触发伪快速重传。利用SACK特别是DSACK可以忽略这个问题。

***

##### 8. 目的度量

TCP能不断学习发送端与接收端之间的链路特征。学习的结果显示为发送端记录一些状态变量，如srtt和rttvar。加入连接关闭，这些状态信息会丢失。而较新的TCP实现维护了这些度量值，即使在连接断开后，也能利用保存的之前的信息作为初始值。

***

##### 9. 重新组包

当TCP超时重传，并不需要完全重传相同的报文段。TCP运行执行重新组包，发送一个更大的报文段来提高性能。