---
layout: post
title:  "Linux环境Nginx安全配置"
categories: Nginx
description: Linux环境Nginx安全配置，Nginx For Ubuntu16.04。
keywords: Nginx, Linux, 安全, Security, Nginx安全
---
# Nginx Security For Ubuntu16.04 #

## 一、隐藏版本号 ##

[nginx爆整数溢出漏洞](http://www.freebuf.com/news/9026.html)
不怕一万，就怕万一。

### 1.1 原始状态 ###

原始情况下，访问错误的链接会有暴露版本信息的危险。

![](http://i.imgur.com/WOYn9rJ.png)

用curl命令查出原始的版本信息：

```shell
curl -I ddos.jinzhucaifu.com
```

![](http://i.imgur.com/vBvNuK2.png)

### 1.2 修改ngx_http_header_filter_module.c文件 ###

进入nginx源码目录，找到`ngx_http_header_filter_module.c`文件进行编辑,找到以下行：

```shell
static char ngx_http_server_string[] = "Server: nginx" CRLF;
```

将nginx修改：

```shell
static char ngx_http_server_string[] = "Server: Yazid" CRLF;
```

### 1.3 修改ngx_http_special_response.c文件 ###

进入nginx源码目录，找到`ngx_http_special_response.c`文件进行编辑,找到以下代码段：

```shell
static u_char ngx_http_error_full_tail[] =
"<hr><center>" NGINX_VER "</center>" CRLF
"</body>" CRLF
"</html>" CRLF
;

static u_char ngx_http_error_tail[] =
"<hr><center>nginx</center>" CRLF
"</body>" CRLF
"</html>" CRLF
;

```

将NGINX_VER去除，将nginx修改：

```shell
static u_char ngx_http_error_full_tail[] =
"<hr><center>""</center>" CRLF
"</body>" CRLF
"</html>" CRLF
;

static u_char ngx_http_error_tail[] =
"<hr><center>Yazid</center>" CRLF
"</body>" CRLF
"</html>" CRLF
;

```

### 1.4 修改nginx.h文件 ###

进入nginx源码目录，找到`nginx.h`文件进行编辑,找到以下代码段：

```shell
#define nginx_version      1011004
#define NGINX_VERSION      "1.11.4"
#define NGINX_VER          "nginx/" NGINX_VERSION
#define NGINX_VAR          "NGINX"
```

将所有敏感信息修改：

```shell
#define nginx_version      0
#define NGINX_VERSION      "0"
#define NGINX_VER          "Yazid/" NGINX_VERSION
#define NGINX_VAR          "Yazid"
```

### 1.5 重新编译 ###

```shell
#编译
root@yazid-chen:/opt/nginx/nginx-1.11.4# make
#安装
root@yazid-chen:/opt/nginx/nginx-1.11.4# make install
#启动
root@yazid-chen:/opt/nginx/nginx-1.11.4# /usr/local/nginx/sbin/nginx
```

### 1.6 隐藏后状态 ###

![](http://i.imgur.com/VpB6yIF.png)

![](http://i.imgur.com/EDN2dFS.png)


## 二、给Nginx以普通用户运行的权限 ##

nginx服务默认启动用户是nobody

![](http://i.imgur.com/lhzo8nw.png)

**1)** 建立nginx用户

```shell
#用户禁止登录，且无家目录
useradd nginx -s /sbin/nologin -M
```

使用id查看用户：

```shell
id nginx
```

**2)** 修改nginx.conf

```shell
user  nobody;
# 将上句修改为
user  nginx;
```

![](http://i.imgur.com/8KL0RsH.png)

**3)** 给关键目录普通用户及组权限

```shell
chown -R nginx conf/ logs/ sbin/
chgrp -R nginx conf/ logs/ sbin/
```

**4)** 普通用户不能通过bind函数绑定小于1024的端口,而root用户可以做到,`CAP_NET_BIND_SERVICE`的作用就是让普通用户也可以绑端口到1024以下的端口。让普通用户启动nginx绑定在80端口：

```shell
setcap CAP_NET_BIND_SERVICE=+ep /usr/local/nginx/sbin/nginx
```

`setcap`来源于`Capabilities`（能力），它打破了UNIX/LINUX操作系统中超级用户/普通用户的概念,由普通用户也可以做只有超级用户可以完成的工作。`setcap`可以设置程序文件的能力。另还有`getcap`可以获得程序文件所具有的能力，`getpcaps`可以获得进程所具有的能力。

**5)** 使用非root用户运行：

```shell
sudo -u nginx /usr/local/nginx/sbin/nginx -c /opt/nginx/nginx-1.11.4/conf/nginx.conf 

```

**6)** 校验启动用户

```shell
ps -aux |grep nginx
```

![](http://i.imgur.com/5lthdxb.png)

## 三、禁用非必要的方法 ##

针对`GET`、`POST`以及`HEAD`之外的请求，如`PATCH`、`TRACE`等，直接返回了`444`状态码（`444`是Nginx定义的响应状态码，会立即断开连接，没有响应正文）。

具体配置是这样的,在`nginx.conf`的`Server`中加入以下代码段：

```shell
if ($request_method !~ ^(GET|HEAD|POST)$ ) {
    return    444;
}
```

可用chrome测试，在nginx的日志`access.log`中看到效果。


## 四、图片防盗链（未测试） ##

配置`nginx.conf`:

```shell
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$ 
{ 
valid_referers none blocked *.epinv.com epinv.com *.qq.com *.baidu.com; 
if ($invalid_referer) { 
  rewrite ^/ http://www.epinv.com/epinv.png; 
  #return 404; 
} 
expires      30d; 
}
```

```shell
#语法
valid_referers none | blocked | server_names | string ...;
```

指定合法的来源`referer`，他决定了内置变量`$invalid_referer`的值，如果`referer`头部包含在这个合法网址里面，这个变量被设置为0，否则设置为1。不区分大小写的。

**参数说明：**

- `none`: 				“Referer” 来源头部为空的情况。
- `blocked`: 			“Referer”来源头部不为空，但是里面的值被代理或者防火墙删除了，这些值都不以http://或者https://开头。
- `server_names`: 		“Referer”来源头部包含当前的server_names（当前域名）。
- `arbitrary string`: 	任意字符串,定义服务器名或者可选的URI前缀.主机名可以使用*开头或者结尾，在检测来源头部这个过程中，来源域名中的主机端口将会被忽略掉。
- `regular expression`: 正则表达式，`~`表示排除https://或http://开头的字符串。

**特别说明：**防盗链跳转的地址，不能再是设置防盗链的虚拟主机地址，要用第三个虚拟主机，要不就成死循环了！


## 五、限制连接请求数（未测试） ##

`ngx_http_limit_conn_module` 可以限制单个IP的连接数，`ngx_http_limit_req_module` 可以限制单个IP每秒请求数，通过限制连接数和请求数能相对有效的防御CC攻击。

### 5.1 限制每秒请求数 ###

`ngx_http_limit_req_module`模块通过漏桶原理来限制单位时间内的请求数，一旦单位时间内请求数超过限制，就会返回503错误。配置需要在两个地方设置：

- nginx.conf的http段内定义触发条件，可以有多个条件。
- 在location内定义达到触发条件时nginx所要执行的动作。

```shell
http {
    limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;//触发条件，所有访问ip限制每秒10个请求
    ...
    server {
        ...
        location /search/ {
            limit_req zone=one burst=5;
        }
```

**参数说明：**

- `$binary_remote_addr`：  	二进制远程地址
- `zone=one:10m`：    		定义zone名字叫one，并为这个zone分配10M内存，用来存储会话（二进制远程地址），1m内存可以保存16000会话
- `rate=10r/s`：     		限制频率为每秒10个请求
- `burst=5`：         		允许超过频率限制的请求数不多于5个，假设1、2、3、4秒请求为每秒9个，那么第5秒内请求15个是允许的；反之，如果第一秒内请求15个，会将5个请求放到第二秒，第二秒内超过10的请求直接503，类似多秒内平均速率限制。
- `nodelay`：         		超过的请求不被延迟处理，设置后15个请求在1秒内处理。

### 5.2 限制IP连接数 ###

设置共享内存区,最大允许连接数对应一个给定的键值。当超过这个极限时,服务器将返回503错误(服务暂时不可用)在回复一个请求:

```shell
http {
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    ...
    server {
        ...
        location /download/ {
            limit_conn addr 1;
        }
```

以上每个IP只允许1个连接。


下面的配置将会限制客户端IP连接到nginx服务器的数量，以及与此同时nginx服务器分发到虚拟服务器的总数：

```shell
http {
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_conn_zone $server_name zone=perserver:10m;
...
server {
    ...
    limit_conn perip 10;
    limit_conn perserver 100;
}
```

设置共享内存区：

```shell
#指令
limit_conn_zone key zone=name:size;
#默认
limit_conn_zone $binary_remote_addr zone=addr:10m;
```

当设置了服务器连接数限制的情况下的日志等级配置，默认error：

```shell
#指令
limit_conn_log_level info | notice | warn | error;

#默认
limit_conn_log_level error;
```

设置拒绝请求时返回的状态码：

```shell
#指令
limit_conn_status code;
#默认
limit_conn_status 503;
```

### 5.3 白名单设置 ###

`http_limit_conn`和`http_limit_req`模块限制了单ip单位时间内的并发和请求数，但是如果Nginx前面有负载均衡或者反向代理，nginx获取的都是来自负载均衡的连接或请求，这时不应该限制负载均衡的连接和请求，就需要`geo`和`map`模块设置白名单：

```shell
geo $whiteiplist  {
        default 1;
        10.11.18.120 0;
    }
map $whiteiplist  $limit {
        1 $binary_remote_addr;
        0 "";
    }
limit_req_zone $limit zone=one:10m rate=10r/s;
limit_conn_zone $limit zone=addr:10m;
```

`geo`模块定义了一个默认值是1的变量`whiteiplist`,并加入白名单IP，其变量`whiteiplist`的值为0。

`whiteiplist`的值对应着`map`模块。1则受限，0则不受限。


## 参考 ##

[Nginx安全优化之隐藏版本号](http://www.bkjia.com/Linux/1124560.html)

[本博客 Nginx 配置之安全篇](https://imququ.com/post/my-nginx-conf-for-security.html#simple_thread)

[nginx documentation](http://nginx.org/en/docs/)

