---
layout: post
title:  "Linux环境Nginx性能优化"
categories: Nginx
description: Linux环境Nginx性能优化，Nginx For Ubuntu16.04。
keywords: Nginx, Linux, 性能, performance, Nginx性能, Nginx performance
---

# Nginx performance optimize#

## 一、主配置文件 ##

### 主配置文件结构 ###

```shell
# 主配置段，也即全局配置段；
···

# 事件驱动相关的配置；
  event {
           ...
         }
   
  # http/https 协议相关的配置段； 
  http {
          ...
    }
   
  # 邮件配置段；
   mail {
          ...
    }
   
  # tcp/udp等相关协议的配置段
   stream {
      ...
    }
```

#### 主配置段相关配置 ####

```shell
##定义执行权限的用户及组，如果省略组，则组名为用户所在组。
#语法：
user user [group];
#默认：
user nobody;

##指定存储nginx主进程编号pid的文件路径
#语法：
pid path;
#默认：
pid logs/nginx.pid;

```

## 二、性能优化 ##

### 2.1 关于CPU的配置项 ###

#### 2.1.1 配置运行处理器 ####

**配置详解：**

```shell
##worker进程的数量，通常应该为当前主机的CPU的物理核心数。
#语法：
worker_processer number;
#默认：
worker_processer  1;

##worker进程绑定到固定的处理器（processer）上
# cpumask(CPU掩码)，八位二进制，每一位代表一个处理器（processer），共8个，1代表固定使用，0代表不使用。
# 00000001 : 0号processer
# 00000010 : 1号processer
# 00000100 : 2号processer
#语法：
worker_cpu_affinity cpumask1 cpumask2 ...;
#默认：
worker_cpu_affinity auto;
```

搞懂上述配置，必须先弄清楚**中央处理器（CPU）**、**内核**、**处理器（processer）**三者的关系。

**CPU**包含一个或多个**内核**，一般一个**内核**包含一个**processer**，而支持一个**内核**有两个**processer**的技术，称为**超线程技术**。此技术可以把**CPU**中一个**核心**模拟成两个用，以实现运算能力的大幅度提升。

**1)** Linux查看CPU信息：

```shell
##总核(cpu cores)数 = 中央处理器（CPU）个数 X 每颗CPU的核数 
##总处理器（processor）数 =总核(cpu cores)数 X 超线程数

#查看CPU总体信息
cat /proc/cpuinfo

# 查看CPU个数
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l

# 查看每个CPU中的核数
cat /proc/cpuinfo| grep "cpu cores"| uniq

# 查看总处理器（processor）个数
cat /proc/cpuinfo| grep "processor"| wc -l
```

通过上述指令，查询到本机仅有1个CPU。

![](http://i.imgur.com/Uet4kiq.png)

该CPU具有4个内核。

![](http://i.imgur.com/ql0WpC5.png)

每核虚拟出2个超线程，作为处理器，共8个处理器。

![](http://i.imgur.com/aOpCeLn.png)

0号和4号处理器同在0号内核上，1号和5号处理器同在1号内核上，以此类推。


**2)** 配置绑定处理器：

`worker_processer 4；`配置处理器数目项最好设置为**小于等于CPU内核数**。

`worker_cpu_affinity 00000001 00000010 00000100 00001000;`绑定处理器项将`worker`进程均分在不同的内核上。

![](http://i.imgur.com/QmDO3Sy.png)

**3)** `./nginx -t` 校验配置项正确性：

![](http://i.imgur.com/XVnd5TH.png)

`./nginx -s reload`重启nginx后，可用以下方法校验：

查看`worker_processes 4;`是否生效：

```shell
ps -aux | grep nginx
```

![](http://i.imgur.com/xqHVZPS.png)

查看`worker_cpu_affinity auto;`是否生效：

```shell
#查看nginx进程在哪个CPU内核上运行
ps -axo pid,user,comm,psr |grep nginx
```

![](http://i.imgur.com/25fLoOI.png)

可以看到，所有的`worker`进程都均匀的分布在不同的内核上，而`master`进程与`pid`为2517的`worker`进程都绑定在0号内核上。其他内核上剩余的处理器共3个，便用作计算机其他进程的处理。

这里有个疑问，0号内核上的`worker`进程和`master`进程同在一个内核会不会给该内核造成极大的负载？

我们先来了解一下`nginx`中`master`和`worker`之间的关系。

正常执行中的`nginx`会有多个进程，最基本的有`master process`（主进程）和`worker process`（工作进程）。`master`充当监控进程，而由主进程`fork()`出来的`worker`则充当工作进程。

![](http://i.imgur.com/22Euu68.png)

`master`进程充当整个进程组与用户的交互接口，同时对进程进行监护。它不需要处理网络事件，不负责业务的执行，只会通过管理`worker`进程来实现重启服务、平滑升级、更换日志文件、配置文件实时生效等功能。

![](http://i.imgur.com/Ouirijn.png)

`worker`进程的主要任务是完成具体的任务逻辑。其主要关注点是与客户端或后端真实服务器（此时nginx作为中间代理）之间的数据可读/可写等I/O交互事件。

`master`进程采用的是信号去通知`worker`进程去做某些工作，这些工作包括`sig_atomic_t ngx_terminate`（强制关闭进程）、`sig_atomic_t ngx_quit`（优雅地关闭进程）、`ngx_uint_t ngx_exiting`（退出进程标志位）、`sig_atomic_t ngx_reopen`（重新打开所有文件）。

如此看来，`master`消耗的资源并不多。

#### 2.1.2 配置进程优先级 ####

```shell
##设定worker进程的nice值，nice值决定进程的优先级，在CPU运行队列较长时，可以被有限调度到CPU上运行。nice值默认为0，区间[-20,20]
#语法
worker_priority number;

```

这里有个疑问：nice值是什么？

首先先看一下进程的类型：

根据消耗型来分，有**I/O消耗型进程**、**CPU消耗型进程**以及两者兼消耗型进程。
**I/O消耗型进程**会把大部分时间消耗在I/O请求和等待I/O上，真正使用CPU的时间很少；而**CPU消耗型进程**会把大部分时间用在使用CPU进行计算等操作。

Linux系统为了给**CPU消耗型进程**多一些处理器时间，而给**I/O消耗型进程**少一些处理器时间，于是linux采取的不是简单的**时间片调度算法**，而是改进的优先级调度算法，**完全公平调度算法CFS**。

我们来执行一个简单的命令：`ps -l`。

![](http://i.imgur.com/f6erFS3.png)

其中有两个参数：**PRI**和**NI**：

- **PRI**即进程的CPU調度优先级，就是进程被CPU执行的先后顺序的数值。它的值是由内核进行动态调整，用户无法调整它的值。此值越小，进程的优先级越高。
- **NI**即nice值，它是一个偏移量。它会影响优先级**PRI(new)=PRI(old)+nice**，范围是-20到+19。root和普通用户所能更改的范围不同，root随意这要在-20-19这个范围内，普通用户0-19。相当于只能增高无法降低。

Linux系统是抢占式的，系统当前运行一个进程，但这个时候一个具有更高优先级的进程突然得到某种资源进入了就绪状态，然后他就来到cpu面前一脚踢开正在运行的进程，与CPU共度美好时光。

至此，我们将`worker_priority`的值设为-5，以提高优先级。

**1）** 查询修改前状态：

```shell
ps axo pid,user,comm,psr,nice | grep nginx
```

![](http://i.imgur.com/VErMiA9.png)

可看出`worker`进程的`nice`值都为0。

**2）** 配置以提高优先级至-5：

```shell
worker_priority -5;
```

`./nginx -s reload` 重启nginx。

**3）** 查询修改后的状态：

![](http://i.imgur.com/MueCDqM.png)

可以看到，`worker`进程的`nice`值已经修改成了-5。



### 2.2 gzip压缩传输数据 ###

`gzip`是若干种文件压缩程序的简称，代表`GNU zip`。

nginx中`gzip`压缩功能由`ngx_http_gzip_module`模块支持。

nginx中`gzip`的主要作用就是用来减轻服务器的带宽问题，经过`gzip`压缩后的页面大小可以变为原来的30%甚至更小，这样用户浏览页面时的速度会快很多。

`gzip`的压缩页面需要浏览器和服务器双方都支持，实际上就是服务器端压缩，传到浏览器后浏览器解压缩并解析。目前的大多数浏览器都支持解析`gzip`压缩过的页面。

```shell
##gzip压缩功能开关
#语法 
gzip on | off;
#默认 
gzip off;

##设置response响应的缓冲区大小。32 4k代表以4k为单位将响应数据以4k的32倍(128k)的大小申请内存。如果没有设置，缓冲区的大小默认为整个响应页面的大小。
#语法 
gzip_buffers number size;
#默认 
gzip_buffers 32 4k | 16 8k;

##设置gzip的压缩级别，可接受的范围是从1到9，数字越大压缩率越高，但更消耗CPU，一般设置6即可。
#语法
gzip_comp_level level;
#默认
gzip_comp_level 1;

##设置允许压缩的页面最小字节数，页面字节数从header头中的Content-Length中进行获取。因为过小的文件内容压缩之后效果不明显，甚至会比不压缩时更大，所以一般建议长度不小于1000或者1K。
#语法
gzip_min_length length;
#默认
gzip_min_length 20;

##指定哪些类型的响应才启用gzip压缩，多个用空格分隔。通配符”*”可以匹配任意类型，不管是否指定”text/html”类型，该类型的响应总是启用压缩。一般js、css等文本文件都启用压缩，如application/x-javascript、text/css、application/xml 等。具体的文件类型对应的mimi-type可以参考conf/mime.types文件。例如 jpg / png 这类文件一般不开启，因为图片格式已经是高度压缩过的，再压一遍没什么效果不说还浪费 CPU
#语法
gzip_types mime-type ...;
#默认
gzip_types text/html;

##设置gzip压缩所需要的请求的最小HTTP版本，低于该版本不使用gzip压缩。一般不用修改，默认即可。
#语法
gzip_http_version 1.0 | 1.1;
#默认
gzip_http_version 1.1;

##如果请求的”User-Agent”头信息能被指定的正则表达式匹配，则对响应禁用gzip压缩功能。主要是为了兼容不支持gzip压缩的浏览器。
#语法
gzip_disable regex ...;
#默认
—

##如果指令gzip,gzip_static或者gunzip被激活的话，启用Vary: Accept-Encoding响应头字段。
#语法
gzip_vary on | off;
#默认
gzip_vary off;

```

我们对nginx做如下配置：

![](http://i.imgur.com/pZ25sfm.png)

保存重启后有：

![](http://i.imgur.com/eUzpULs.png)

而未开启gzip压缩的同一页面，其响应大小及时间如下：

![](http://i.imgur.com/qQuBvDu.png)

我们再来看一下响应头信息：

![](http://i.imgur.com/n1qAMnb.png)

可以看到，响应头中已经存在`Content-Encoding:gzip`，并且有返回我们设置的Vary值`Vary:Accept-Encoding`。

在响应头中我们还看到`Transfer-Encoding:chunked`。

**1)** `Transfer-Encoding`字面意思是**传输编码**，而`Content-Encoding`字面意思是**内容编码**。

- `Content-Encoding`通常用于对实体内容进行压缩编码，目的是优化传输，例如`gzip`压缩，能大幅减小体积。
- `Transfer-Encoding`则是用来改变报文格式的，它不但不会减少实体内容传输大小，甚至还会使传输变大。

HTTP协议中有一个重要概念：`Persistent Connection`（持久连接，即**长连接**）。**长连接**的存在，主要是可以避开缓慢的三次握手，还可以避免遇上TCP慢启动的拥塞适应阶段。

**长连接**通过`Connection: keep-alive`这个头部来实现，服务端和客户端都可以使用它告诉对方在发送完数据之后不需要断开TCP连接，以备后用。HTTP/1.1则规定所有连接都必须是持久的，除非显式地在头部加上 `Connection: close`。

对于非持久连接，浏览器可以通过连接是否关闭来界定请求或响应实体的边界；对于持久连接，通过`Content-Length`的长度信息，判断出响应实体已结束。那如果`Content-Length`和实体实际长度不一致时，则由`Transfer-Encoding`来解决。

在头部加入`Transfer-Encoding: chunked`之后，就代表这个报文采用了**分块编码**。这时，报文中的实体需要改为用一系列分块来传输。每个分块包含十六进制的长度值和数据，长度值独占一行，长度不包括它结尾的`CRLF（\r\n）`，也不包括分块数据结尾的`CRLF`。最后一个分块长度值必须为0，对应的分块数据没有内容，表示实体结束。

**2)** Nginx中如果启用了`gzip`压缩，则必然采用`Transfer-Encoding:chunked`的方式输出，原因如下：

在Nginx内部，`r->headers_out.content_length_n`用于表述请求返回内容的长度，只有在`r->headers_out.content_length_n >=0`的时候，才有意义。如果没有在脚本中强制`header`输出`content-length`，则默认在Nginx中`r->headers_out.content_length_n = -1`。

在Nginx中，`header`输出和`body`的输出是完全两个不同的阶段，`http header`输出是由`ngx_http_send_header()`执行，由各个功能模块调用。`body`的输出由`ngx_http_writer()`调用`ngx_http_output_filter()`产生。

`header`和`body`都有一个`filter`队列需要执行，分别是`ngx_http_top_header_filter`和`ngx_http_top_body_filter`，在http的功能模块中把处理函数插入到这两个队列。

真正的输出动作由`ngx_http_write_filter()`产生。所以`filter`队列的顺序很重要，`ngx_http_write_filter`应该处于队列最后。也就是先输出`header`，再输出`body`。

由此可见，`gzip`要对内容模块进行压缩处理，而在`header filter`的时候，`gzip`模块不可能计算出压缩后的内容长度。所以Nginx会清空`header`中的`content-length`。便采用`Transfer-Encoding:chunked`的方式输出。

既然`gzip`是内容编码，则压缩是在传输之前进行的，所以传输的分块是按照压缩后的数据分块的。


## 参考 ##

[nginx源码解析(4)-深入http模块](http://blog.liwenxin.com/2010/11/25/nginx-code-reading-4.html)

[Response与Transfer-Encoding:chunked、Content-Length、Content-Encoding:gzip](https://phpor.net/blog/post/2037)

[HTTP 协议中的 Transfer-Encoding](https://imququ.com/post/transfer-encoding-header-in-http.html)

