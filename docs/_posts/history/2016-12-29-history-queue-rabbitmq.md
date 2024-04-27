---
title: RabbitMQ 的高可用
date: 2016-12-29 13:47 +0800
excerpt: "RabbitMQ 是一个队列服务，用 Erlang 实现，大多用在对可用性要求极高的场景中"
categories:
- history
- MQ
- RabbitMQ
feature_image: "/assets/static/img/content.jpg?image=872"
---

** Introduction
   一个朋友说我的生活里工作占用的时间比例太大了，几乎完全没有生活，
   可能我早就被痛苦和工作、学习奴役了吧，只是无脑的为了学习而学习，谁知道呢，知道的越少越好吧，与其明白自己生活在苦难中后
   让自己更痛苦，还不如完全不理会，所谓“身在福中不知福”，那我就“身在苦中不知苦”好了。

   最近对 RabbitMQ 的高可用做了一些了解，包括 KeepAlived 的原理和配置过程，简单记录、分享一下。

   对了，标题我尝试用英文来写了，最近因为英文被一个陌生人轻轻的鄙视了一下，宝宝表示很不开心，暂时用英文写标题，以后
   大概有一天可以用英文来写正文部分了吧(这是一个美好的愿望)。

** What's the RabbitMQ
   简单说，当然了，我了解的也不够多，事实上我在尽力往复杂了说。

    RabbitMQ 是一个队列服务，用 Erlang 实现，大多用在对可用性要求极高的场景中。
    它有这么几个概念：exchanges、queues、channels，但事实上我没有深入了解，猜测应该和 Kfaka 的 Topic 类似吧，
    类似分组，比如每个 channel 可以有多个 queue，每个 exchange 可以有多个 channel。

    嗯，是的，介绍完了，就酱，介绍完了，不过，正题是 HA 对吗？所以还是进(tao)入(bi)主题吧。

** Highly Availability in RabbitMQ
*** Unsynchronised Slaves
   和所有的高可用集群服务一样，当你的数据进入一个队列时，数据会被复制到其他的 Slave 节点。
   它可以在不停止服务的情况下，在线将 Slave 加入集群中，在加入集群后，它不会包含任何数据，
   当有新数据进入集群队列的时候，数据会被同步到新增的这个节点，数据一直在被消费，直到集群中
   的所有“旧数据”(新节点中不存在的数据)被消费光的时候，新节点才会和整个集群的数据保持完全同步。
*** Mirroring
    这些名词真是够了，明明是一样的东西，非得叫不同的名字，比如这个 RMQ 中，它的集群关系叫“镜像”，
    把数据“镜像”到 Slave。

**** Mirror policy
    数据镜像有几种策略：
    | ha-mode | ha-params  | Result                                                                                                                                                                   |
    |---------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
    | all     | (absent)   | 队列会镜像到集群中的所有节点，当新节点加入进来的时候队列会被镜像到新节点(如上解释)                                                                                       |
    | exactly | count      | 队列镜像到指定数量的节点中，如果集群中现有节点少于指定的节点，队列会被镜像到所有节点。否则，多余的节点会在有节点宕机时才会启用进入该集群开始同步，啊，不，开始镜像数据。 |
    | nodes   | node names | 用“实例名”来指定一个节点列表，当实例名包含在这个列表里时，数据才会镜像过来。节点名默认使用“rabbit@${HOSTNAME}，当集群中没有其他可用节点时，这些没有被包含进来的节点才会启用开始镜像数据。                                                              |


**** Append a node to cluster
     #+BEGIN_SRC shell
       # 1) master:
       rabbitmqctl stop_app
       rabbitmqctl reset
       rabbitmqctl start_app
       # 2) other node:
       rabbitmqctl stop_app
       rabbitmqctl reset
       # 这里的 --ram 参数指定了将数据保存在内存中，如有需要可以去掉，默认会写到硬盘上。
       rabbitmqctl join_cluster --ram rabbit@rabbitmq1
       rabbitmqctl start_app
     #+END_SRC


**** Configuration the policy
     #+BEGIN_SRC shell
       # 有三种方式来配置 node 策略:
       # 1. CLI:
       # 这里支持正则，第三个参数是匹配 queue name，
       # 你的队列名，第四个参数就是上边提到的策略
       # rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}'
       # 2. HTTP API:
       # PUT /api/policies/%2f/ha-all {"pattern":"^ha\.","definition":{"ha-mode":"all"}}
       # 3. Web UI:
       # RMQ 可以开启 WEB 管理界面，
       # 通过 WEB 界面也可以配置，这里就不介绍了。
     #+END_SRC
** End
   好了，就到这里了，基本上这就是关于 RMQ 集群的核心概念了，看起来今天已经很晚了，
   我还是考虑把 KeepAlived 的内容分开放到下一篇好了。