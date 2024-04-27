---
title: SaltStack 速览
date: 2017-01-07 21:19
excerpt: "常用模块介绍"
categories:
- history
- ops
- automation
- saltstack
feature_image: "/assets/static/img/content.jpg?image=872"
---

### Introduction
   本来没准备写关于 Salt 的东西，总觉得对它的了解太少了，后来听了一次中国用户组发起人组织的一个沙龙，发现我了解的东西和这些大咖讲的内容还是比较接近的，差距没有那么大，但仍然有很多人听的入神，现在准备把这些内容拿出来写写，刚好给公司同事做一个简单的培训。
### SALT ESSENTIALS
   [SaltStack 官方文档](https://docs.saltstack.com/en/latest/)

   __建议按照以下排名顺序了解__
   1. [Running Commands](https://docs.saltstack.com/en/latest/topics/execution/remote_execution.html)
   2. [Targeting](https://docs.saltstack.com/en/latest/topics/targeting/)
   3. YAML
      * Master/Minion 配置文件
   4. [Grains](https://docs.saltstack.com/en/latest/topics/grains/)
      * Minion 端定义
      * Master 端定义
   5. [Pillar](https://docs.saltstack.com/en/latest/topics/pillar/)
      * Master 端定义
   6. [State](https://docs.saltstack.com/en/latest/ref/states/)
      * SLS 文件
   7. Modules
      * Built-in
      * Custom
   8. [Runners](http://salt-zh.readthedocs.io/en/latest/ref/runners/)
   9. [Returner](https://docs.saltstack.com/en/latest/ref/returners/)
   10. [Best Practices](https://docs.saltstack.com/en/latest/topics/best_practices.html)

### Running Commands
   这部分主要用来说明命令语法，通常谈到命令行控制操作默认都在说命令“salt”
   - Usage
      salt [options] '<target>' <function> [arguments]
   - Examples

   ``` shell
      $ sudo salt '*' test.ping
      ali-bj-xbb-web-main-02:
      True
      ali-bj-xbb-web-main-01:
      True

      $ sudo salt '*' cmd.run 'echo Hello World'
      ali-bj-xbb-web-main-02:
      Hello World
      ali-bj-xbb-web-main-01:
      Hello World
   ```

### Targeting
   - Target Type
      1. 通配符
      2. 正则
      3. 分组
      4. 列表
      5. Grains
      6. Pillar
   - Examples
      * 通配
         ``` shell
            $ sudo salt '*-web-*' test.ping
            ali-bj-xbb-web-main-02:
            True
            ali-bj-xbb-web-main-01:
            True
         ```
      * 正则匹配
         ``` shell
            $ sudo salt -E 'ali-bj-xbb-web-main-[0-9]*' test.ping
            ali-bj-xbb-web-main-02:
            True
            ali-bj-xbb-web-main-01:
            True
         ```
      * 列表匹配
         ``` shell
            $ sudo salt -L 'ali-bj-xbb-web-main-01,ali-bj-xbb-web-main-02' test.ping
            ali-bj-xbb-web-main-02:
            True
            ali-bj-xbb-web-main-01:
            True
         ```
      * Grains 匹配
         ``` shell
            $ sudo salt -G 'os:CentOS' test.ping
            ali-bj-xbb-web-main-02:
            True
            ali-bj-xbb-board-main-01:
            True
            ali-bj-xbb-file-public-01:
            True
            ali-bj-xbb-web-main-01:
            True
            ali-bj-xbb-preweb-main-01:
            True
         ```
      * Pillar
         同 Grains
      * 混合
         ``` shell
            salt -C 'G@os:Ubuntu and minion* or S@192.168.50.*' test.ping
         ```
      * Batch size
         #+BEGIN_SRC sh
         $ sudo salt '*' test.ping -b 2
         #+END_SRC
### States
   该模块是 SaltStack 做配置管理方面的核心模块，能够对 Minions 节点的状态做编程，从编程角度来统一管理需求、状态、环境等不一致。

   [States 文档](https://docs.saltstack.com/en/latest/topics/tutorials/starting_states.html)

   - Example
      * Top file
         __该文件的作用是在一开始定义一个机器列表和要执行的操作的映射__
         ``` yaml
            base:
               '*':
            - basepkg
               'web-*':
            - xbbweb
               'os:CentOS' :
            - match: grain
         ```
      * [Action sls](https://docs.saltstack.com/en/latest/ref/states/all/)
         ``` yaml
            basepkg:
               pkg.installed:
               - name: vim
                  vim_config:
               file.managed:
               - source: salt://config/vimrc
               - name: /root/.vimrc
         ```

### Salt Grains
   __Salt comes with an interface to derive information about the underlying system. This is called the grains interface.__

   以上是官网对它的解释，本人觉得 Grains 完全可以理解成机器属性，这个属性是静态的，并且是可以自定义的，salt 默认会读取一些值用来被使用，也可以用户自己添加一些值，可以在 Minion 端添加，也可以在 Master 端添加。

   __该属性是 Key － Value 形式存在的。__

   - Example
      * show grains
         1. 列出所有 Grains key
            ``` shell
               salt '*' grains.ls
            ```
         2. 列出所有键值对
            ``` shell
               salt '*' grains.items
            ```
      * Grains in the minion config

         在 /etc/salt/minion 配置中添加以下配置

         ``` yaml
            grains:
               roles:
               - webserver
               - memcache
               deployment: datacenter4
               cabinet: 13
               cab_u: 14-15
         ```

      * 获取静态配置的 grains 值
         ``` shell
            # 注，grains.get 命令用于获取单个 grains key 的内容
            $ sudo salt '*-board-*' grains.get roles
            ali-bj-xbb-board-main-01:
            - webserver
            - memcache
         ```
   - WRITING GRAINS
      动态获取某些状态用来赋值给 minion 的 grains

      Custom grains modules should be placed in a subdirectory named _grains located under the file_roots specified by the master config file. The default path would be /srv/salt/_grains.

      1. 自定义模块(直接赋值)
         ``` python
            def yourfunction():
               # initialize a grains dictionary
               grains = {}
               # Some code for logic that sets grains like
               grains['yourcustomgrain'] = True
               grains['anothergrain'] = 'somevalue'
               return grains
         ```
      2. 自定义模块(动态的状态)

         ``` python
            import socket
            def yourfunction():
                  grains = {}
                  hostname = socket.gethostname()
                  grains['custom_hostname'] = hostname
                  if '-web-' in hostname:
                     grains['service_type'] = 'web'
                  elif '-cache-' in hostname:
                     grains['service_type'] = 'cache'
                  return grains
         ```

### Salt Pillar
   __Pillar is an interface for Salt designed to offer global values that can be distributed to minions. Pillar data is managed in a similar way as the Salt State Tree__

   上边 Grains 和 Pillar 的功能类似，都是在 minion 端定义一些属性，但是 Grains 是在 minion 端定义的，而 pillar 是在 master 端定义且保存的，在使用时需要指定 minion 来将特定的 pillar 的值赋值给 minion，也就是说，除了指定 minion 外，其他节点都会变得这些信息，所以可以选择保存一些安全类相关的信息。

   - Example
      * Master 开启 Pillar
         ``` yaml
            # /etc/salt/master
            pillar_roots:
               base:
               - /srv/pillar
         ```
      * 指定目录中配置 pillar
         1. 以下配置为顶级文件，必须命名为 top.sls
            ``` yaml
               # /srv/pillar/top.sls
               base:
               '*-web-*':
               - packages
            ```
         2. 配置其中指定的值
            {% raw %}
            ``` yaml
               # /srv/pillar/packages.sls
               {% if grains['os'] == 'RedHat' %}
               apache: httpd
               git: git
               {% elif grains['os'] == 'Debian' %}
               apache: apache2
               git: git-core
               {% endif %}
               passwd: 123qwe
            ```
            {% endraw %}
      * 在 Grains 中调用
         ``` yaml
            apache:
               pkg.installed:
               - name: {{ pillar['apache'] }}
                  auth_to_cmdb:
               cmd.run:
               - name: /bin/bash xxxx --password {{ pillar['password'] }}
         ```

### Salt Runners
   __可以在 master 执行模块，功能类似 master 上安装 minion 后对其当作 minion 执行操作。__
   * Examples
      ``` python
         import salt.client

         def up():
            '''
            Print a list of all of the minions that are up
            '''
            client = salt.client.LocalClient(__opts__['conf_file'])
            minions = client.cmd('*', 'test.ping', timeout=1)
            for minion in sorted(minions):
               print minion
      ```
   * Full list of runner modules
      | name       |     Description                                                            |
      |------------|----------------------------------------------------------------------------|
      | cache      | Return cached data from minions                                            |
      | doc        | A runner module to collect and display the inline documentation from the   |
      | fileserver | Directly manage the salt fileserver plugins                                |
      | jobs       | A convenience system to manage jobs, both active and already run           |
      | launchd    |                                                                            |
      | manage     | General management functions for salt, tools like seeing what hosts are up |
      | network    | Network tools to run from the Master                                       |
      | search     | Runner frontend to search system                                           |
      | state      | Execute overstate functions                                                |
      | virt       | Control virtual machines via Salt                                          |
      | winrepo    | Runner to manage Windows software repo                                     |