---
title: 基于 Virtualbox 虚拟机的跨平台开发环境
date: 2016-12-20
excerpt: "这个方式已经没必要了，早就可以用 docker 来实现了，可能是当时的构思"
categories:
- history
- vm
- virtualbox
- env
feature_image: "/assets/static/img/content.jpg?image=872"
---
### 目标
一键安装，（几乎）不提供任何文档（只提供一个 http 可执行文件下载地址），环境（基础组件全部安装并开机启动）安装后即可用PHP开发，共享本机代码目录（自定义），所有操作使用安装包来指导完成，至少保证 Mac、Windows 平台可以不需要任何背景知识即可安装,*地址固定*，仅宿主机可连接，IP 默认固定为

宿主机虚拟网卡：
- 10.1.1.10/24

虚拟机网卡地址：
- Name: vboxnet0
- IP: 10.1.1.100/24 gw 10.1.1.10
- dns1: 172.18.10.252
- dns2: 114.114.114.114

---


### 流程

- 安装 Virtualbox 虚拟机
- 利用 Pyvbox 模块来对虚拟机网络做全局配置
   * 新建虚拟网卡 vboxnet0（目前已知 Windows 系统下会自动建立本网卡）
   * 配置其 DHCP IP 范围（如果使用静态地址则忽略此步骤）
- 修改本机配置：
   * Linux：
      ``` shell
      sysctl -w net.ipv4.ip_forward=1
      iptables -t nat -A POSTROUTING -s 10.1.1.0/24 -j MASQUERADE
      ```
   * Mac：
      ``` shell
      sysctl -w net.ipv4.ip_forward=1
      # 获取有 IP 地址的网卡
      # 这些配置添加到 /etc/pf.conf
      nat on {en0, enX} proto {tcp, udp, icmp} from 10.1.1.0/24 to any -> {en0, enX}
      pass from {vboxnet0, 10.1.1.0/24} to any keep state
      # 添加后执行：
      sudo pfctl -e -f /etc/pf.conf
      ```
   * Windows：

      将宿主机的当前可使用的物理网卡选项 “允许其他网络用户通过此计算机的 Internet 连接来连接”
- 下载并导入虚拟机模板
- 根据提示配置IP及共享目录（IP可自定义，或者使用默认地址）