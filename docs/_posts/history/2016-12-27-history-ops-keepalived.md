---
title: 深入了解 Keepalived
date: 2016-12-27 17:32 +0800
excerpt: "KeepAlived 的 VIP 功能的实现则是基于 VRRP 协议的"
categories:
- history
- vrrp
- keepalived
feature_image: "/assets/static/img/content.jpg?image=872"
---

### Introduction
   1. Netfilter & Iptables
   2. LVS
   3. VRRP
   4. KeepAlived

   今天的主题是 KeepAlived，但是在它之前，还是得普及一些其他知识，否则就只剩它的配置文件了，那样就没激情了，也就不能四射了，哇哈哈哈。

   简单简单简单，接下来的这些内容都会尽量简单并且直接的描述，我们的目的是大概了解工作原理，或者说工作模型，方便查问题以及整理思路，能快速设计自己的架构并且将自己的预期可靠并且快速的实现。

   今天的主题是 KeepAlived 这个工具，但核心更多地是和网络知识相关的，KeepAlived 的出现事实上是为了解决 LVS 的单点问题，LVS 是一个工作在 3 / 4 / 7 层中的代理工具，可以根据不同的策略代理请求分发给后端的服务器。KeepAlived 虽然可以只用来提供 VIP，但它的出现是为了解决 LVS 这个代理的单点问题，所以即使我们只用到了它的 VIP 的功能，仍然要了解 LVS 的原理，因为它才是让 KeepAlived 发挥更好的作用的地方。

   而 LVS 是基于 Netfilter 这个框架实现的，所以必须要介绍 Netfilter。

   KeepAlived 的 VIP 功能的实现则是基于 VRRP 协议的，所以这里我需要介绍 Netfilter 、LVS 的调度策略、
   VRRP 协议的规范，最终会了解 KeepAlived 的适用场景。

### Netfilter & Iptables
   一个数据包的正常流转，或者说我们认识到的正常流转流程应该是，从网卡接收到数据，
   直到将数据包解包后处理完数据接着封装数据从网卡出去，或者将数据转发给其他地址。

   Netfilter 是 Linux 内核中数据包流转模型的框架，数据包在整个流转过程中会经过几个特殊的点，而
   Netfilter 的作用则是在这些特殊的点去调用一些回调函数，而这些函数就是数据包的处理规则。

   而 iptables(可能已经过时，貌似 RHEL 早有了新的解决方案来替换 iptables) 则是一些预定义的规则，
   可以通过 iptables 来将这些规则注册到到 netfilter 的回调函数中。

   (这里应该有画外音，因为需要画图，所以我又去找 dot 的文档了，然后用 dot 来画图，嘿嘿嘿~)

   [[/images/netfilter.png][这张图是用来描述数据包在内核中的生命周期]]

   途中红色和蓝色分别代表两种不同方向的数据流，当内核从网卡拿到数据后首先经过 prerouting 这个
   检查点， 如果目的地是本地的，则是“数据流 B”，数字则是处理步骤， userspace 指的是应用层处理数据
   之后，数据发往 output 检查点；如果数据流是需要转发的，则通过红色 “数据流  A” 的方向流转。

   有了这个图，接下来对 LVS 就好理解的多了。

### LVS(Linux Virtual Server)
*** How it works
   在看过了 Netfilter 模型之后，来理解下 LVS 的工作原理，上图：
   [[/images/lvs-netfilter.png][这张图描述了 LVS 的数据流转路径   ]]

   绿色的“数据流 C”路径便是 LVS，LVS 在 input 检查点注册了一些行为按照不同的转发策略
   将直接将其送往 forward 检查点，随后出网将数据分发给后端的 real server。

   因为 VIP 是本地（LVS 服务器）的一个独立 IP，所以请求始终会进入 INPUT 这个注册点，或者
   叫链，所以 LVS 会在这里注册需要执行的回调函数。

   这里有一点非常清楚了，各个发行版为了安全，一般都会把系统的内核允许转发数据的规则配置
   不允许转发，所以，当你要配置、使用它的时候，必须要允许内核转发数据包。

*** Transfer mode
    我们知道了它的工作原理，转发策略有三种，下面来简单介绍一下，为了方便解释过程，会忽略公网的部分。
**** DR（direct）
     该策略叫 direct，顾名思义，就是直接转发，将收到的数据包直接发给后端的服务器。
     流程大概是这样：
     1. 客户端发起广播，查找拥有目的地址（VIP）的 MAC 地址
     2. LVS 回应请求，然后接收到客户端的请求数据
     3. LVS 将目的地 MAC 地址改为后端 real server 的 MAC 地址
     4. 将修改了 MAC 地址的数据发送给 real server
     5. readl server 处理请求，并且和客户端直接通信
     6. 断开连接

     在整个数据交互过程中，有几个前提必须要说明
     1. LVS 和 real server 必须在同一广播域
     2. real server 必须也拥有 VIP
     3. real server 不能对广播到 VIP 的请求做响应

     为什么要有这样几个前提呢？DR 模式的优势是让客户端和 real server 直接通信，避免由 LVS 的瓶颈导致
     降低性能，甚至服务不可用。当客户端发起广播请求时，拥有同样 IP 的 real server 不能对请求做响应，
     否则就不能得到由 LVS 来达到代理的目的。

     所以，real server 在配置了 VIP 的同时，要在内核参数里修改 arp_ignore 、arp_announce 等几个参数，
     要达到的目的是，不对请求 VIP 的广播做出响应，“除非你是直接发给我的”(就像 LVS 所做的，将目的
     MAC 地址直接改为 real server 的 MAC 地址)，因为这个模式 LVS 修改的是目的地址的 MAC 地址，所以
     如果它和 real server 不在同一个广播域的话，请求是不可能 到达 real server 的。

**** NAT（Network Address Translation）
     其实这个模式应该算是最简单的了，因为在日常生活中也能看到或者需要用到很多这样的例子，
     呃。。或者说叫“日常工作中”，只是简单的做 DNAT、SNAT，所有的数据都经过 LVS 服务器，客户端
     和 NAT 之间保持连接，LVS 和 real server 之间保持连接，LVS 维护连接并且转发数据。

     这个模式的好处是，只要 LVS 和客户端直接畅通，LVS 和 real server 之间畅通，服务便可用，
     劣势也很明显，那就是 LVS 可能成为瓶颈， NAT 的特点就是频繁的数据包头的修改源、目的地址，
     并且要维护这些连接，而不是把维护这些连接的任务分配给后端的 real server。

**** TUN（tunnell）
     这个模式相比 DR 理解起来也轻松一些，应该算是一个说简单也不简单，说不简单也没有那么复杂。

     为什么我说的这么抽象呢，因为这个模式就像它的名字一样，用到了隧道的方式。

     该模式特点：
     1. real server 直接和客户端维护连接
     2. real server 拥有 VIP
     3. LVS 和 real server 之间维护一个 ip 隧道，real server 需要安装隧道客户端

     这个模式里，当数据请求到达 LVS 时，LVS 将整个数据包打包，然后在数据包之上再打包一层 IP
     数据，源是 LVS 自己，目的地是 real server，然后将数据发出，real server 接收到数据后，解包
     将 LVS 打包的隧道数据变成原始数据来处理，将隧道数据中的源、目的变为本次通信的源、目的
     地址，然后回复给客户端。

**** comparison
     简单的总结一下三种转发模式的区别，

     DR 优势：性能，既效率最高
     DR 劣势：LVS 和 real server 必须在同一广播域，网络比较复杂的环境可能无法使用

     NAT 优势：网络没有任何限制，对于网络复杂的场景比较适合
     NAT 劣势：性能最差

     TUN 优势：性能几乎可以媲美 DR，网络没有限制
     TUN 劣势：需要 ipip 隧道支持，可能对操作系统有些要求，除 Linux 系统外，其他系统如果要使用可能有些麻烦。

*** Scheduling algorithm
    了解了它有这些工作模式，接下来就该讨论它是用这些工作模式来干什么

    LVS 一共有10种调度算法：
    1. RR(round-robin)
       该模式均衡的把请求分发给后端的每一台 real server，很好理解，不再多说。
    2. WRR(Weighted Round-Robin)
       每个 rea l server 都可以配置权重值，LVS 调度器会根据每一台 real server 的权重值
       来分发请求，具体过程为：
       比如，这里有服务器 A、B、C，权重值分别为 4、3、2，那么它的调度模式为：
       AABABCABC

       请求会按照你配置的权重值，将权重值大的首先得到任务，然后权重值（逻辑的）减1，
       直到多台看 real server 权重值相同后，再将其分发给其他 real server，如此反复执行。

       /*该模式可以根据不同服务器的性能来进行调度*/

    3. LC(Least-Connection)
       最小连接数调度，请求来之后，从 real server 中选出一台连接数最小的节点去分发请求。
    4. WLC(Weighted Least-Connection)
       通过 连接数 / 权重值 得到结果最小的服务器选为下一个调度地址
    5. LBLC(Locality-Based Least-Connection)
       局部性的最小连接，该模式通常用于缓存调度，并且在代理多个服务的情况下更加突出其特点，
       假设所有 real server 的权重或其他调度依赖的值都相同，则把该服务的相同的目标 IP 的请求
       调度到同一个 real server。
    6. LBLCR（Locality-Based Least-Connection with Replication）
       The locality-based least-connection with replication scheduling algorithm is also for destination IP load balancing. It is usually used in cache cluster. It differs from the LBLC scheduling as follows: the load balancer maintains mappings from a target to a set of server nodes that can serve the target. Requests for a target are assigned to the least-connection node in the target's server set. If all the node in the server set are over loaded, it picks up a least-connection node in the cluster and adds it in the sever set for the target. If the server set has not been modified for the specified time, the most loaded node is removed from the server set, in order to avoid high degree of replication.
    7. DHS(Destination Hashing Scheduling)
       基于目的地址 hash 的调度
       貌似和上条相同，谈到了 cluster，这个概念还不太理解，以后再议
       The destination hashing scheduling algorithm assigns network connections to the servers through looking up a statically assigned hash table by their destination IP addresses.
    8. SH (Source Hashing)
       这个比较好理解了，源地址的 Hash， 和 Nginx 的 IP Hash 一样的。
    9. SED(Shortest Expected Delay)
       The shortest expected delay scheduling algorithm assigns network connections to the server with the shortest expected delay. The expected delay that the job will experience is (Ci + 1) / Ui if sent to the ith server, in which Ci is the number of connections on the the ith server and Ui is the fixed service rate (weight) of the ith server.
    10. NQS(Never Queue)
	The never queue scheduling algorithm adopts a two-speed model. When there is an idle server available, the job will be sent to the idle server, instead of waiting for a fast one. When there is no idle server available, the job will be sent to the server that minimize its expected delay (The Shortest Expected Delay scheduling algorithm)

     这个没什么好说的了，根据自己不同的场景使用不同的调度策略吧。

** VRRP(Virtual Router Redundancy Protocol)
*** Introduction
    目前我们了解了 LVS 的实现原理，了解了 LVS 的调度方式，接下来该了解 KeepAlived 了，不对不对，
    了解 KeepAlived 之前还得了解它的实现原理，下面先简单介绍一下 RFC 中对 VRRP 的主备节点不同的
    启动过程中执行的操作。

    VRRP 协议事实上是一个路由协议，就像标题所说，虚拟路由冗余协议，它的作用是解决路由单点问题，
    例如两台路由之间互联，并且同时连接其他网络，这两台设备之间一台主一台备，同时只有一台设备在
    启用，两台设备之间有心跳检测机制，如果备份设备检测到主不可用时，则将两台设备共享的 IP 抢过来，
    告知其他本广播域内的设备共享 IP，更准确的是说虚拟 IP 的新 MAC 地址，从而达到设备的热备份。

*** How it works
     英语何其重要啊，还记得多年前有记者采访 Linus 大神还是 Stallman 大神来着，大意是“你这东西怎么
     写出来的啊”？然后对方回答，很简单啊，“去读协议，然后编码，就行了。”

     我想骂娘了，可是我特么的，读英语有多困难啊？ POSIX、RFC 这些，我特么怎么看？？

     好吧，话说回来， VRRP 只解释 IPV4 的，IPv6 我看不懂，╭(╯^╰)╮

     [[https://tools.ietf.org/html/rfc5798#page-20][地址在这里，想看就来吧]]

**** Initialize
     本段描述了当一个 vrrp 协议初始化时所做的事情。

     if  如果接收到一个启动事件（就是接收到其他设备的 vrrp 启动时）{
	if 优先级为 255 的拥有 VIP {
	    1. 广播 VIP --> 新 MAC 的信息
	    2. 修改状态为 Master
	} else {
	    1. 设置两个检查 Master 状态的 Timer
	    2. 将自己的状态设置为 Backup
	}
     }

**** Backup
     本节描述 Backup 节点如何确保 Backup 状态如何以及监控 Master 节点状态的。

     while {
	 1. 必须不回应来自 VIP 的 ARP 请求
	 2. 必须丢弃所有目的地为 VIP 的 MAC 地址的数据包
	 3. 不接收任何来自 VIP 的数据包
	 if 接收到一个 shutdown 事件时(自己被 shutdown 的时候) {
	     1. 取消健康检查
	     2. 转变为 初始化 状态
	 }
	 if master 宕机次数/时间到 {
	     1. 设置 Adver_Timer (健康检查间隔)
	     2. 将自己的状态变更为 Master
	 }
	 if 收到一条广播 {
	     if 广播优先级为0 {
		 1. 设置 Master 的过期时间（在 KeepAlived 中可以自定义）
	     } else 优先级非 0 {
		 if Backup 的夺权模式未开启 或者 优先级高于本地 {
		     1. 设置 master 过期时间
		     2. 重新计算 master 失效时间
		     3. 重置 master 失效时间（为指定时间，keepalived 中自定义）
		 } else Backup 的夺权模式开启 或者 对方优先级高于本地 {
		     1. 丢弃广播包
		 }
	     }
	 }
     }
**** Master
     本节描述了 Master 状态在各个场景中都做了什么

     while {
	 1. 响应（VIP的） ARP 请求
	 2. ... 处理所有 VIP 地址的数据（类似废话省略）
	 if 接收到 shutdown 事件（如果自己被 shutdown 了） {
	     1. 取消健康检查
	     2. 广播自己优先级变为  0
	     3. 转为 初始化 状态
	 }
	 if 接收到 vrrp 广播数据 {
	     if 优先级为0 {
		 1. 重置健康检查事件
	     } else {
		 .......
	     }
	 }
     }

**** End
     好吧，算了，好像协议描述这么具体可能也没有什么意义，嗯。。。大概就是这样了，
     总而言之，就是主备之间各种健康检查机制以及优先级状态来保证同广播域内只有一个
     设备可用，并且如果主宕机，备份设备会恢复状态来提供服务。

** KeepAlived
*** Introduction
    KeepAlived 的核心是用了 VRRP 的协议来保证服务（包括路由服务）的热备。而它出现的目的则是
    为了解决 LVS 的代理单点问题，在这一点上，如果它不是为了 LVS 而出现的话，
    可能就没有必要将它和 LVS 放在一起来说，因为它出现的目的，致使它不仅在程序中
    实现了 VRRP 协议的功能，而且还在其中实现了直接调用 LVS 模块来达到对 LVS 这个
    服务的配置。

*** How to use it
    如果你在某度去搜索它的使用方式的话，一般会找当一种说法就是
    “它有两种使用方式，一是配合 Nginx，二是配合 LVS”

    呵呵呵~事实上它确实可以有两种实现方式，但是 Nginx 没有半毛钱的关系。

    1. 只提供 VIP 来保证当前服务器的业务状态
       1) 在 KeepAlived 配置文件中不配置后端节点
       2) 定义针对本地服务的健康检查的方式（比如 Nginx，某度搜出来的一般都是这种方式）
	  当选择这么做的时候，可以定义在本地状态非健康时，将优先级降低，
	  而健康的那个节点的优先级则会高于另一台，这样做始终保证了可用的那一台设备
	  的优先级高于另一台
    2. 结合 LVS 来做代理分发请求，使用这个方式后，将提供 VIP 及请求分发两种功能
       如果使用该方式，就要配置每一个后端节点的健康状态检查方式，从而也不用直接对
       LVS（ipvsadm）的命令做任何操作，只要 KeepAlived 配置文件写好，LVS 也就配置好了。

** End
    KeepAlived 也就介绍到这里了，基本上所有知识点都涉及到了，具体配置的话，百度一搜一大把，
    或者官网，基本上有了以上背景知识外，基本上看一下就都知道配置什么意思了，排错应该也不会
    有什么问题，and good luck。