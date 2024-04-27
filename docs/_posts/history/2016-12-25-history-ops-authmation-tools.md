---
title: 运维工具简介、区别
date: 2016-12-25 10:21
excerpt: "常见任务编排工具的对比"
categories:
- history
- automation
- tools
feature_image: "/assets/static/img/content.jpg?image=872"
---
### Puppet
   - 优势
      1. Strong compliance automation and reporting tools
      2. 社区活跃
      3. WEB 平台，近实时的节点管理
      4. 和 SHELL 结合很好
      5. 安装简单并且支持多平台
      6. 针对大企业级用户有实用、稳定并且成熟的解决方案
   - 缺点
      1. 新用户需要学习 Puppet DSL 或者 Ruby
      2. 安装出错排错难，没有完善的错误报告能力
      3. 括容复杂，DSL 配置会变得非常复杂
      4. 缺乏同时系统（例如 salt 的文件服务器）
      5. 企业版仅支持 <10 节点
### Chef
   - 优势
      1. 中间件和操作系统的管理最灵活的解决方案之一
      2. 为程序员打造
      3. 文档很好，社区活跃
      4. 起始，看了第二条就没必要继续看了
      5. 需要付费

### Ansible
   - 优势
      1. 简单的远程执行功能，登录方便（这条更像一个跳板系统的功能）
      2. 适用于快速发展的环境
      3. Shares facts between multiple servers, so they can query each othe（不懂）
      4. 安装简单
      5. 新用户学习成本低
      6. push、pull 模形都支持
      7. 无需安装 agent
      8. 可付费获取支持
   - 缺点
      1. 太过强调“命令编排”，甚至大过了配置管理本身
      2. SSH慢
      3. 需要 root 权限以及 Python 环境（这就是所谓的不需要客户端）
      4. 组件之间语法不同（比如赖以成名的 playbook 和 模板）
      5. （几乎）没有  UI

### Salt
   - Pors
      1. 高弹性
      2. 简单、直观的安装及配置
      3. Strong introspection.
      4. 社区活跃
      5. 丰富的扩展（貌似只有 salt 有这条），yaml 语法的配置
      6. 最初开发目的是用于监控，所以比如进程监控类的角度赖看也很强大
   -  Cons
      1. 新用户安装可能有点麻烦
      2. 文档多，但是管理不太好，查找不太容易
      3. （几乎）没有 UI
      4. 除 Linux 外的其他操作系统来说不是最好的选择