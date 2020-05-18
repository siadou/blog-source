---
title: WebRTC技术原理
date: 2020-05-18 17:15:44
tags: [前端技术, P2P]
---

> 最近在用Sip.JS实现网络软电话的功能，于是抽时间了解了下WebRTC的相关原理。

WebRTC (Web Real-Time Communications) 是一项实时通讯技术，它允许网络应用或者站点，在不借助中间媒介的情况下，建立浏览器之间点对点（Peer-to-Peer）的连接，实现视频流和（或）音频流或者其他任意数据的传输。WebRTC包含的这些标准使用户在无需安装任何插件或者第三方的软件的情况下，创建点对点（Peer-to-Peer）的数据分享和电话会议成为可能。

WebRTC是一个完全对等技术，用于实时交换音频、视频和数据，同时提供一个中心警告。如其他地方所讨论的，必须进行一种发现和媒体格式协商，以使不同网络上的两个设备相互定位。这个过程被称为信令，并涉及两个设备连接到第三个共同商定的服务器。通过这个第三方服务器，这两台设备可以相互定位，并交换协商消息。

常见应用包括：SIP协议实现web电话，web torrent，视频会议等。

# ICE框架协议栈
Interactive Connectivity Establishment (ICE)是一个允许你的浏览器和对端浏览器建立连接的协议框架(不是一个协议)。在实际的网络当中，有很多原因能导致简单的从A端到B端直连不能如愿完成。这需要绕过阻止建立连接的防火墙，给你的设备分配一个唯一可见的地址（通常情况下我们的大部分设备没有一个固定的公网地址），如果路由器不允许主机直连，还得通过一台服务器转发数据。

ICE协议只是制定规范，没规定怎么实现细节，Google的WebRTC是ICE的实现。

ICE整合了STUN、TURN，通过使用以下几种技术完成上述工作：

## NAT
网络地址转换协议Network Address Translation (NAT) 用来给局域网设备映射一个公网的IP地址的协议。一般情况下，路由器的WAN口有一个公网IP，所有连接这个路由器LAN口的设备会分配一个私有网段的IP地址（例如192.168.1.3）。私网设备的IP被映射成路由器的公网IP和唯一的端口，通过这种方式不需要为每一个私网设备分配不同的公网IP，但是依然能被外网设备发现。

一些路由器严格地限定了部分私网设备的对外连接。这种情况下，即使STUN服务器识别了该私网设备的公网IP和端口的映射，依然无法和这个私网设备建立连接。这种情况下就需要转向TURN协议。

## STUN
NAT的会话穿越功能Session Traversal Utilities for NAT (STUN) (缩略语的最后一个字母是NAT的首字母)是一个允许位于NAT后的客户端找出自己的公网地址，判断出路由器阻止直连的限制方法的协议。
STUN服务器位于公网上并且有一个简单的任务：检查传入请求的IP和端口地址（来自在NAT网络中运行的应用程序）并将该地址作为响应发回。换句话说，应用程序使用STUN服务器查询其位于公网上的IP和端口。此过程使WebRTC端点能够查询到自己公开访问的地址，然后通过信令机制将其传递给另一个端点，以便建立直接链接。（事实上，不同的NAT以不同的方式工作，并且可能存在多个NAT层，但原理仍然是相同的）。
客户端通过给公网的STUN服务器发送请求获得自己的公网地址信息，以及是否能够被（穿过路由器）访问。

![](/images/web-rtc/stun.png)


## TURN
一些路由器使用一种“对称型NAT”的NAT模型。这意味着路由器只接受和对端先前建立的连接（就是下一次请求建立新的连接映射）。TURN是对STUN的扩展。

NAT的中继穿越方式Traversal Using Relays around NAT (TURN) 通过TURN服务器中继所有数据的方式来绕过“对称型NAT”。你需要在TURN服务器上创建一个连接，然后告诉所有对端设备发包到服务器上，TURN服务器再把包转发给你。很显然这种方式是开销很大的，所以只有在迫不得已的情况下采用。

TURN与STUN的共同点都是通过修改应用层中的私网地址达到NAT穿透的效用，异同点是TURN采用了两方通讯的“中间人”方式实现穿透，突破了原先STUN协议无法在两台主机不能够点对点直接连接下提供作用的限制。

![](/images/web-rtc/turn.png)

## SDP
会话描述协议Session Description Protocol (SDP) 是一个描述多媒体连接内容的协议，例如分辨率，格式，编码，加密算法等。所以在数据传输时两端都能够理解彼此的数据。本质上，这些描述内容的元数据并不是媒体流本身。

从技术上讲，SDP并不是一个真正的协议，而是一种回话描述格式，用于描述在设备之间共享媒体的连接。

## SDP报文格式
SDP描述由许多文本行组成，文本行的格式为<类型>=<值>，<类型>是一个字母，<值>是结构化的文本串，其格式依<类型>而定。详见在RFC 2327。

**＜type＞=<value>[CRLF]**

一个SDP会话描述包含如下部分:
1. 会话名称和会话目的
2. 会话的激活时间
3. 构成会话的媒体(media)
4. 为了接收该媒体所需要的信息(如地址,端口,格式等)
因为在中途参与会话也许会受限制,所以可能会需要一些额外的信息:
1. 会话使用的的带宽信息
2. 会话拥有者的联系信息

![](/images/web-rtc/sdp.png)


### SDP交换模型 —— Offer/Answer模型

上文说到,SDP用来描述多播主干网络的会话信息,但是并没有具体的交互操作细节是如何实现的,因此RFC3264定义了一种基于SDP的offer/answer模型.在该模型中,会话参与者的其中一方生成一个SDP报文构成offer,其中包含了一组offerer希望使用的多媒体流和编解码方法,以及offerer用来接收改数据的IP地址和端口信息.offer传输到会话的另一端(称为answerer),由answerer生成一个answer,即用来响应对应offer的SDP报文.answer中包含不同offer对应的多媒体流,并指明该流是否可以接受.

RFC3264只介绍了交换数据过程,而没有定义传递offer/answer报文的方法,一种实现是RFC3261/SIP即会话初始化协议中描述。

## ICE工作流程


ICE的基本思路是,每个终端都有一系列传输地址(包括传输协议,IP地址和端口)的候选,可以用来和其他端点进行通信.
其中可能包括:
1. 直接和网络接口联系的传输地址(host address)
2. 经过NAT转换的传输地址,即反射地址(server reflective address)
3. TURN服务器分配的中继地址(relay address)

虽然潜在要求任意一个Peer的候选地址都能用来和Peer的候选地址进行通信。但是实际中发现有许多组合是无法工作的.举例来说,如果A和B都在NAT之后而且不处于同一内网,他们的直接地址就无法进行信.ICE的目的就是为了发现哪一对候选地址的组合可以工作,并且通过系统的方法对所有组合进行测试(用一种精心挑选的顺序).

![](/images/web-rtc/ice.png)

如图所示，A想与B进行通信，过程如下：
1. A收集所有的IP地址，并找出其中可以从STUN服务器和TURN服务器收到流量的地址；
2. A向STUN服务器发送一份地址列表，然后按照排序的地址列表向B发送启动信息，目的是实现节点间的通信；
3. B向启动信息中的每一个地址发送一条STUN请求；
4. A将第一条接收到的STUN请求的回复信息发送给B；
5. B接到STUN回复后，从中找出那些可在A和B之间实现通信的地址；
6. 利用列表中的排序列最高的地址进一步的设备间通信。

由于该技术是建立在多种NAT穿透协议的基础之上，并且提供了一个统一的框架，所以ICE具备了所有这些技术的优点，同时还避免了任何单个协议可能存在的缺陷。因此，ICE可以实现在未知网络拓扑结构中实现的设备互连，而且不需要进行对手配置。另外，由于该技术不需要为VoIP流量手动打开防火墙，所以也不会产生潜在的安全隐患。



# 实体

## 会话描述
WebRTC连接上的端点配置称为会话描述。
该描述包括关于要发送的媒体类型，其格式，正在使用的传输协议，端点的IP地址和端口以及描述媒体传输端点所需的其他信息的信息。 使用会话描述协议(SDP)来交换和存储该信息; 如果您想要有关SDP数据格式的详细信息，可以中找到。

## 信令
信令用于协调通信，WebRTC应用开始通话之前，客户端需要交换一些信息（信令）：
用于打开或关闭通信的会话控制消息，如：
1. 错误信息。
2. 媒体元数据，例如编解码器和编解码器设置，带宽和媒体类型。
3. 用于建立安全连接的的秘钥信息。
4. 主机的IP和端口等网络信息。
（以上部分信息以SDP形式传递）为了避免冗余并提高与已有技术的兼容性，WebRTC标准未规定信令方法和协议。JavaScript会话建立协议（JSEP）描述了一种大致的方法：

WebRTC没有规定信令层的协议。这是因为不同的应用程序可能更喜欢使用不同的信令协议，比如已经存在的SIP或者Jingle信令协议，抑或一些针对应用定制的协议。在这种方法中，需要交换的关键信息是多媒体会话描述（SDP），它指定了建立媒体连接所必需的传输和媒体配置信息。

JSEP需要在 offer / 提议 和 answer / 应答 的点与点之间交换上文提到的媒体元数据信息。交换信息的两个点之间使用SDP会话描述协议进行通信。SDP协议消息格式大概是这个样子：


常用信令扩展：
XMPP（可扩展消息传递和呈现协议）：基于XML为即时消息传递开发的可用于信令的协议。
开源库，如ZeroMQ和OpenMQ。NullMQ使用基于WebSocket的STOMP协议将ZeroMQ概念应用于Web平台。
使用WebSocket的商业云消息传递平台，例如Pusher，Kaazing和PubNub。
商业WebRTC平台，如vLine。


## 信令服务器

两个设备之间建立WebRTC连接需要一个信令服务器来实现双方通过网络进行连接。
信令服务器并不需要理解和解释信令数据内容。通过信令服务器的消息的内容实际上是一个黑盒。重要的是，交换两点之间传递的信令。内容对其来讲并不重要。
client到信令服务器需要双向通信，所以最简单的实现是websocket。其他可以采用轮询、长轮询等方法。
信令服务器不传递Media信息。

## ICE候选

除了交换关于媒体的信息(上面提到的Offer / Answer和SDP )中，对等体必须交换关于网络连接的信息。 这被称为ICE候选者，并详细说明了对等体能够直接或通过TURN服务器进行通信的可用方法。 通常，每个同伴将首先提出最佳候选人，让他们走向更糟糕的候选人。 理想情况下，候选地址是UDP(因为速度更快，媒体流能够相对容易地从中断恢复 )，但ICE标准也允许TCP候选。


# 通信过程

RTCPeerConnection是WebRTC应用程序在点对点之间创建连接并传送音频和视频的API。
要想创建音视频通信连接，RTCPeerConnection有两个任务：

1. 确定本地媒体信息，例如分辨率和编解码器信息。这是用于offer和answer机制的元数据。
2. 获取应用程序主机的网络地址，称为candidate。

![](/images/web-rtc/connection.png)


offer/answer过程：
1. Alice创建了一个RTCPeerConnection对象。
2. Alice使用RTCPeerConnection的createOffer()方法创建了一个offer（一个SDP会话描述文本）。
3. Alice调用setLocalDescription()将她的offer设置为本地描述。
4. Alice把offer转换为字符串，并使用信令机制将其发送给Eve。
5. Eve对Alice的offer调用setRemoteDescription()函数，为了让他的RTCPeerConnection知道Alice的设置。
6. Eve调用createAnswer()函数创建answer。
7. Eve通过调用setLocalDescription()将自己的answer设置为本地描述。
8. Eve使用信令机制把自己字符串化的的answer传给Alice。
9. Alice使用setRemoteDescription()函数将Eve的answer设置为远程会话描述。

Alice和Eve也需要去交换网络信息。“查找候选地址candidate”一词是指使用ICE框架查找网络接口和端口的过程。

1. Alice创建RTCPeerConnection对象的时候会生成一个onicecandidate句柄。
2. 这个句柄在网络candidate生效时会被调用。
3. Alice通过信令通道将字符串化的candidate数据发送给Eve。
当Eve从Alice获取candidate消息时，她调用addIceCandidate()，将candidate添加到远程对等描述中。


![](/images/web-rtc/connection-full.png)




# 参考文献

[WebRTC API](https://developer.mozilla.org/zh-CN/docs/Web/API/WebRTC_API)

[WebRTC中的信令和内网穿透技术 STUN / TURN](https://blog.csdn.net/shaosunrise/article/details/83627828)

[P2P通信标准协议(三)之ICE](https://www.cnblogs.com/pannengzhi/p/5061674.html)

[P2P技术详解(三)：P2P技术之STUN、TURN、ICE详解](https://www.cnblogs.com/mlgjb/p/8243690.html)