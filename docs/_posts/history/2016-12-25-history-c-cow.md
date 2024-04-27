---
title: Linux进程的写时复制
date: 2016-12-25
excerpt: "用实验证明 写时复制"
categories:
- history
- C语言
- 写时复制
feature_image: "/assets/static/img/content.jpg?image=872"
---

今天偶然看到一篇文章，标题是《Linux，你拿什么证明自己的写时复制？》，看了看标题，我就在想，对啊，我可
真是太懒了，一直都知道它 fork 子进程的时候用得是“COW”的技术，可以，我是真的没亲自求证过，今天就测试下好了。

### 什么是“COW”？

   COW（Copy on write：写时复制）大概是这样的概念，每个进程都有自己的代码段、数据段等等各种段，在一个独立的进程中
   所有的内容都是仅该进程独享的，可以毫无顾虑的对数据做任何修改。
   那么在创建一个子进程时，会把父进程的所有数据做一个映射表，将父进程的数据都映射到子进程中，当子进程要读取某些数据的
   时候，会通过映射表而读取父进程中的保存数据的地址，但是当它要修改某些数据的时候，地址会发生改变，会把修改后的数据
   保存在新的地址中。

### 什么是 fork ？

   *nix 系统中都提供了一个 API —— fork，这个函数的作用是创建一个子进程，man 手册中是这么介绍的
   “Fork() causes creation of a new process.  The new process (child process)
   is an exact copy of the calling process (parent process)”
   它会有两个返回值，确切的说是两次返回值，一次是在父进程中返回的，它的返回值是子进程的 id，另一次是在子进程中
   的返回值，它的值会是0.

### 实验证明它的写时复制代码

   以下是测试代码，这里有一个全局的数据 “data”，它保存了一个默认值——10，之后分别在子进程和父进程中都将其
   修改为不同的值，同时在修改前和修改之后分别都对打印了各自的数据以及数据地址。

   ``` C
      #include <stdio.h>
      #include <unistd.h>

      char *data = {"10"};

      void child()
      {
      printf("data in child before modify: %s\n", data);
      printf("data address in child before modify: %p\n", data);
      data = "20";
      printf("data in child after modify: %s\n", data);
      printf("data address in child after modified: %p\n", data);
      }

      int main(void)
      {
      if (0 == fork()) {
            child();
      } else {
            printf("data in father before modify: %s\n", data);
            printf("data address in father before modify: %p\n", data);
            data = "30";
            printf("data in father after modified: %s\n", data);
            printf("data address in father after modified: %p\n", data);
      }
      return 0;
      }
   ```

### 测试结果

   下面是运行结果（这里还要提醒一下，fork 之后先运行子进程还是先运行父进程完全看内核的心情，这次示例中是
   先运行了子进程），显而易见，在父/子进程中修改值之前，它们的值和内存地址是一模一样的，
   而在修改之后它们的地址发生了变化。在子进程中已经将数据修改，可是父进程中的原始数据还是没有发生变化，
   当父进程自己修改了数据才能看到地址变了。

   ``` shell
     ##############################
     data in child before modify: 10
     data address in child before modify: 0x10e857e3c
     data in child after modify: 20
     data address in child after modified: 0x10e857ea9
     ##############################
     data in father before modify: 10
     data address in father before modify: 0x10e857e3c
     data in father after modified: 30
     data address in father after modified: 0x10e857f61
   ```

### 题外话
   其实今天的主题是硬插进来的，因为我弄了一个博客，我想宣传一下，哈哈哈，自娱自乐什么的挺好的。