---
title: Ganglia 的自定义监控项
date: 2016-12-25 10:17
excerpt: "Ganglia一个分布式监控，很强大"
categories:
- history
- C语言
- monitor
feature_image: "/assets/static/img/content.jpg?image=872"
---

在我了解 Ganglia 这个监控之前，一直在思考一个问题，如果有很多很多节点，每个节点都有很多很多监控指标的情况下， 普通的监控应该怎么实现呢？后来用了 NSCA，确实解决了一部分问题，但问题还是存在。

对于 Ganglia 这个词，我是在今年夏天第一次听到的，当时在一家朋友内推的公司面试，面试官提到了这个词，但是我当时根本听不懂，不知道怎么写，然后就默默的记了下来，以为这是一个新出的流行的监控，后来当我了解了以后才知道，原来这东西已经流行快20年了。

By the way, 很抱歉后来没去那家公司，当时啥都聊好了结果我反悔了，让那个朋友很难堪，真的抱歉。。。

### What's the Ganglia
它是一个分布式监控，对分布式，比起我之前了解的分布式模型的基于 NSCA 的 Nagios 来说，要强大的多，但是它俩也不冲突，Ganglia 更擅长绘图、取数据，而 Nagios 更多的是用于设置各种阈值来达到报警的目的，所以，你应该可以想到了，它俩可以集成在一起，Nagios 从 Ganglia 取数据，然后确定达到什么条件执行什么策略。

Ganglia 是 UC Berkeley 发起的一个开源集群监视项目，为了解决大规模集群的监控问题，它的核心包括两个进程 gmond、gmetad，以及一个前端显示曲线图的  Web 程序（ PHP 实现）。

- gmond：这个进程的作用是在每个客户端来接受/发送数据
- gmetad：统计节点，在 Ganglia 服务器端用来将集群中从 gmond 获取到的数据绘图

### 监控中的小问题思考
仔细想了想，太不擅长做报告了，大家都懂，PPT 是一般报告的主要呈现载体，而 PPT 最好的部分一定是图，用一张图来让观看者一眼就能明白你想表达的意图。其实我的这个文档也是，如果会画图的话就太好了，不过。。。着实不会。

先抛出一个问题吧

- 如果你有一个大的集群池（假定：共一千个节点的集群）
- 这个集群池里又分不同服务类型的池子（5个集群池，每个集群池200个节点）

一般的监控怎么实现呢？对了，在这里我们只谈客户端需要装一个客户端的模型，对于不需要装客户端的模型一般都是
小场景，我们这里就不谈了。

传统监控实现方式：
- Nrpe
   __监控服务端 -> 客户端__
- NSCA
   __客户端 -> 监控服务器__

不管以上哪种方式，都是需要在每台客户端和一台监控服务器连接，然后送/取数据达到监控的目的。如上例所说，如果有 1, 000 个节点，那么就需要所有客户端和服务器执行 *连接->数据->断开* 这样的过程。显而易见，1, 000 个节点中有可能单台的数据就有 1, 000  条，对监控服务器的性能要求非常高，对网络带宽的消耗也非常非常大。

这里其实包含两个问题
- 监控服务器是否能够迅速处理完所有的请求
- 监控服务器有多少节点就得向多少节点发送（Nrpe）或者接收（NSCA）多少次的数据

### Ganglia 的特点
为了解决上节中提到的第二个问题，Ganglia 在网络层得到了很好的解决——组播，不知道大家对组播的概念了解多少，只要是在支持组播的网络中， 即使有一万个节点，Ganglia 只要发送一条数据就可以利用组播的特性让每一个需要接收的数据的节点接收到该数据。

上节中第二个问题是处理数据的性能，Ganglia 支持一种我理解的“池中池”的方式，因为它本身就是为了大规模集群而出现的解决方案，所以呢。。。我想想怎么画图

__多个最小节点 -> 多个gmond -> 多个gmond -> gmetad__

每个小集群中的节点将数据发送到多个（为了实现高可用）gmond 中，之后可以选择继续发送到其他 gmond中，或者发送给 gmetad 来处理数据。

通过这两个优势点，我们解决了上节中发现的传统监控中的问题。

### 监控项
   默认的 Ganglia 支持的监控项有 CPU、 磁盘、网络、IO、流量等数据，然后这一点上和其他所有监控服务一样，会有很多需要自己来实现的针对自己的场景来扩展的监控项。

   Ganglia 默认支持 Python、Perl 等模块，很无奈，本人对 Perl 一点都不懂，它虽然古老，但是也一直想学，不过又有点叶公好龙了，从来没实施过学习的行为，不谈了。对于 Python 来说，算是我比较擅长的一个语言了，但是呢，它默认只支持 Python 2.6 以上 3.0 以下的版本，虽然由于 2 和 3 版本间的差异导致大家很难接受 3，甚至还听说有一些小伙伴转去其他语言了，不过唯独我这个变态钟爱 3，既然不支持 3，那我只好转向其他语言了（果然变态吧？）。

   C，嗯，由于 Ganglia 本身是用 C 来实现的，所以对 C 的支持很好，现在最高版本是 3.7.2，不过据说是 3.1 以后才支持自定义扩展的。所以，接下来说说它的 C 模块怎么写。

### C Modules
   这里是它的 [Ganglia Wiki](https://github.com/ganglia/monitor-core/wiki/Ganglia-GMond-C-Modules) 地址，我就不一一翻译了，简单介绍下。
   - 模块类型
      * 动态
      * 静态

      这两种模块的区别是，静态模块需要写死你有哪些监控项，名字叫什么，动态模块可以通过读取你的配置文件来选择监控哪些数据。

      静态监控项很好理解了，比如你要监控内存使用了多少，只要写好了，它叫 “Memory USED”就可以了。

      动态监控项稍微解释下，比如要监控某个进程的进程数，也许你今天要监控 “php-fpm” 有多少个，明天还要监控 “bash” 有多少个，那就不能写死了，你的需求应该是在配置文件里指定这个要监控的项目叫什么，可以随便增加。

   - 模块的结构
      - 数据结构
      这里是按照 WIKI 里的示例还写的，我写的模块就不拿出来了 ^_^

      首先要定义一个数据结构，类型是 mmodule，结构名叫 mem_module

      ``` c
         mmodule mem_module =
         {
            STD_MMODULE_STUFF,
            mem_metric_init,
            mem_metric_cleanup,
            mem_metric_info,
            mem_metric_handler,
         };
      ```

      然后按照这个数据结构里以 mem_metric 开头的函数分别实现，这里的 mem_metric 是可以取名叫其他的，这里并没有限制。

      这几个函数简单介绍如下：

      1. STD_MMODULE_STUFF: WIKI 解释说，每个模块都要定义一个这个，如果你需要改的话就不需要看这个 WIKI 页面了。（我觉得挺逗逼。。。就不能好好说话吗？解释下会死啊？）
      2. mem_metric_init: 必须定义的函数 (static int metric_init(apr_pool_t) *)，这个函数是被 GMOND 第一个调用的函数。
      3. mem_metric_cleanup: 必须定义的函数 (static void metric_cleanup(void))，这个函数是在从 GMOND 卸载该模块的时候调用的，通常什么都不需要做，如果有什么必须在卸载该模块的时候做的事情的话，在这里做就好了。
      4. mem_metric_info: 必须定义的结构，该结构会在调用 metric_init 的时候对这里的数据进行初始化。
      5. mem_metric_handler: 必须定义的函数，这个函数会在 metric_init 中对 metric_info 中的每一个数据项执行一次。

      - mem_metric_info

         ``` C
            static Ganglia_25metric mem_metric_info[] = {
               {0, "mem_total",  1200, GANGLIA_VALUE_FLOAT, "KB", "zero", "%.0f", UDP_HEADER_SIZE+8, "Total amount of memory displayed in KBs"},
               {0, "mem_free",    180, GANGLIA_VALUE_FLOAT, "KB", "both", "%.0f", UDP_HEADER_SIZE+8, "Amount of available memory"},
               {0, "mem_shared",  180, GANGLIA_VALUE_FLOAT, "KB", "both", "%.0f", UDP_HEADER_SIZE+8, "Amount of shared memory"},
            }
         ```

         这里需要定义你的监控项的元数据，每一列的解释如下：

         1. int key: 我真的觉得这 WIKI 页面很逗笔，对这一项的解释是：我不清楚这一项是干啥的，但是
         设置为 0 看起来没错。
         2. char *name: 监控项的名字
         3. int tmax: maximium time in seconds between metric collection calls，不翻译了，
         关心本篇文章的应该都能看的懂
         4. Ganglia_value_types type: 定义数据类型，用于 APR（Ganglia 使用了 Apache 的 APR
         实现的加载 .so 模块，前边忘介绍了）为 metric 动态创建存储数据类型。可以是其中之一：
         string, uint, float, double or ganglia Global.
         5. char *units： RRD 图显示的单位
         6. char *slope: one of the zero, positive, negative, or both
         7. char *fmt： 格式化
         8. int msg_size: 数据长度
         9. char *desc: 监控项说明

      - mem_metric_init

         ``` C
            static int mem_metric_init ( apr_pool_t *p ) {
               int i;
               libmetrics_init();
               for (i = 0; mem_module.metrics_info[i].name != NULL; i++) {
                  MMETRIC_INIT_METADATA(&(mem_module.metrics_info[i]), p);
                  MMETRIC_ADD_METADATA(&(mem_module.metrics_info[i]),MGROUP,"memory");
               }
               return 0;
            }
         ```

         足够简单了吧？在循环中初始化数据。

         1. MMETRIC_INIT_METADATA 执行初始化
         2. MMETRIC_ADD_METADATA 将调用我们的 handler 将初始化好的数据结构和结果配个对儿 ^_^

      - mem_metric_handler

         ``` C
            static g_val_t mem_metric_handler ( int metric_index )
            {
               g_val_t val;
               switch (metric_index) {
                  case 0:
                  return mem_total_func();
                  case 1:
                  return mem_free_func();
                  case 2:
                  return mem_shared_func();
               }
               val.f = 0;
               return val;
            }
         ```

         在调用 handler 的时候，会按 info 中定义顺序来依次调用，并且按照序号将值传递进来。忘记强调下了，handler 要返回一个 g_val_t 结构的数据，要用 g_val_t->f 这个指针来保存
你的数据结果。g_val_t 结构定义如下：

         ``` c
            typedef union {
               int8_t   int8;
               uint8_t  uint8;
               int16_t  int16;
               uint16_t uint16;
               int32_t  int32;
               uint32_t uint32;
               float   f;
               double  d;
               char str[MAX_G_STRING_SIZE];
            } g_val_t;
         ```

### 使用模块
    以下是从我写的 Makefile 和配置文件修改了下贴上来的，不保证完全可配合上边程序可用，
    只是示例喔 ^_^
   - Makefile

      ``` shell
         GANGLIAROOT = /path/to/ganglia-3.7.2
         GANGLIABIN=/path/to/ganglia-3.7.2/lib64/ganglia
         APRINCLUDE = /usr/include/apr-1
         SYSLIBS = /usr/lib64
         INCLUDE = -I$(GANGLIAROOT)/libmetrics -I$(GANGLIAROOT)/include -I$(GANGLIAROOT)/lib -I$(APRINCLUDE)
         LIBS = -L$(SYSLIBS)
         OPTS = -Wall -std=c99 -lcurl
         cc = gcc

         mod_mem_test.so:
            $(cc) $(INCLUDE) $(LIBS) $(OPTS) --shared -o mod_nginx_status.so -fPIC mod_mem_test.c
         default:
            mod_mem_test.so
            cp mod_mem_test.so $(GANGLIABIN)
         clean:
            rm -rf *.so
            rm -rf *.o
            rm -rf *~
      ```

     这部分显然我写的很不专业，一直没有很认真的研究过 Makefile ，所以凑乎着用吧

#### 配置文件
   - conf

      ``` shell
         modules {
               module {
                  name = "nginx_mem_test"
                  path = "mod_mem_test.so"
                  params = "/path/to/ganglia-3.7.2/lib64/ganglia/"
            }
         }

         collection_group {
            collect_every = 30
            time_threshold = 30
            metric {
               # 这里注意， name 要求和 metric_info 中定义的 char *name 一样
               name = "mem_total"
               # 这一项是 RRD 图中的描述信息，随意写
               title = "Test for totally memory"
            }
         }
      ```

### 结尾

    好了，大功告成，动态模块我这里就不介绍了，也不复杂。

    开头我说过了 Ganglia 和 Naigos 其实可以结合起来一起用，让 Nagios 从 Ganglia 来取数据，并且
    可以在 Naigos 页面中添加 “小太阳” 来查看每一台服务器的每一个监控项的 RRD 图，回头等我都
    换完了踩踩坑，然后再来分享下 ^_^