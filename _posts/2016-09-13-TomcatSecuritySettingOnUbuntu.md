---
layout: post
title:  "Linux环境Tomcat安全配置"
categories: Security
description: Linux环境Tomcat安全配置，Tomcat For Ubuntu16.04。
keywords: Tomcat, Linux, 安全, Security, Tomcat安全
---
# Tomcat Security For Ubuntu16.04 #

## 一、权限 ##

### 1.1 创建tomcat用户 ###

为了提高系统安全，tomcat不应该使用root运行。为它创建一个新用户和组。
创建一个tomcat组：

```
sudo groupadd tomcat
```

创建一个叫tomcat的用户：

```
sudo useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat
```

tomcat用户属于tomcat组，家目录是`/opt/tomcat`，我要把tomcat安装在这个目录。`/bin/false`代表这个用户是不能登录的。

### 1.2 更改权限 ###

赋给tomcat用户各种权限：

```
cd /opt
sudo chgrp -R tomcat tomcat

cd /opt/tomcat
sudo chgrp -R tomcat conf
sudo chmod g+rwx conf
sudo chmod g+r conf/*

```

修改各种目录的所有者：

```
cd /opt
sudo chown -R tomcat tomcat

cd /opt/tomcat
sudo chown -R tomcat webapps/ work/ temp/ logs/ bin/ lib/
```

### 1.3 配置定时登出 ###

如果使用启用了Tomcat用户，则需要设置Tomcat定时登出，将`$CATALINA_HOME\conf\server.xml`配置如下：

```
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

## 二、配置开机启动 ##

我们需要把tomcat配置为服务，为了做到这一点，需要创建systemd服务配置文件。

使用下面命令查看Java安装路径：

```
sudo update-java-alternatives -l
```

现在在`/etc/systemd/system`目录创建服务文件tomcat.service：

```
sudo vim /etc/systemd/system/tomcat.service
```

tomcat.service内容如下：

```
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target
 
[Service]
Type=forking
 
Environment=JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64/jre
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINA_BASE=/opt/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'
 
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
 
User=tomcat
Group=tomcat
 
[Install]
WantedBy=multi-user.target
```

替换JAVA_HOME的值，注意在路径后加jre；上面配置内存要根据需要修改。

修改完成之后，重新加载systemd：

```
sudo systemctl daemon-reload
```

启动tomcat：

```
sudo systemctl enable tomcat
sudo systemctl start tomcat
```

确认tomcat启动状态：

```
sudo systemctl status tomcat
```

![](http://i.imgur.com/wD55zo0.png)


## 三、安全加固配置 ##

### 3.1 删除应用包 ###

除了需要部署上去的应用，其余位于`$CATALINA_HOME\webapps`文件夹中的应用如docs、examples、host-manager、manager和ROOT，若无业务必要，请执行删除上述的应用包。

```
rm -rf docs/ examples/ host-manager/ manager/ ROOT/
```

### 3.2 禁止Tomcat目录列表 ###

确保`$CATALINA_HOME\conf\web.xml`中listings的值为false：

```
<init-param>
    <param-name>listings</param-name>
    <param-value>false</param-value>
</init-param>

```

### 3.3 设置Cookie的HttpOnly属性 ###

如果您在cookie中设置了HttpOnly属性，那么通过js脚本将无法读取到cookie信息，这样能有效的防止XSS攻击。

在`$CATALINA_HOME\conf\context.xml`文件中添加`useHttpOnly="true"`配置如下：

```
<Context useHttpOnly="true">
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
    <WatchedResource>${catalina.base}/conf/web.xml</WatchedResource>
</Context>

```

进入项目路径找到web.xml：

```
root@yazid-chen:/opt/tomcat/webapps/api/WEB-INF# vim web.xml 

```

加入http-only配置：

```
<session-config>
      <session-timeout>30</session-timeout>
      <cookie-config>
            <http-only>true</http-only>
            <secure>true</secure>
      </cookie-config>
</session-config>
```

### 3.4 配置shutdown端口 ###

在`$CATALINA_HOME\conf\server.xml`中有`“<Server port="8005" shutdown="SHUTDOWN">”`的配置。

任何人只要telnet到服务器的8005端口，输入"SHUTDOWN"，然后回车，服务器立即就被关掉了。

从安全的角度上考虑，需要把这个shutdown指令改成一个别人不容易猜测的字符串。而且这个修改不影响shutdown.bat或shutdown.sh的执行。

配置如下：

```
<Serverport="未被占用的端口" shutdown="较为复杂的字符串">
#注：配置的端口需要大于1024。
```

### 3.5 隐藏Tomcat版本信息 ###

在默认配置下，当应用出现异常时，客户端会显示Tomcat的版本信息。攻击者可以根据Tomcat版本信息选择漏洞库攻击，所以需要将Tomcat的版本信息隐藏，解压`$CATALINA_HOME\lib\catalina.jar`：

```
root@yazid-chen:/opt/tomcat/lib# jar xf catalina.jar 

```

将`\org\apache\catalina\util`中的配置ServerInfo.properties如下,info和number随意：

```
server.info=Server
server.number=Y
server.built=Jul 4 2016 18:22:47 UTC

```

### 3.6 关闭war自动部署 ###

默认的配置war放在`$CATALINA_HOME\webapps`中会自动部署，所以关闭war自动部署防止被植入木马等恶意程序。将`$CATALINA_HOME\conf\server.xml`配置如下：

```
<Host name="localhost"  appBase="webapps"
      unpackWARs="false" autoDeploy="false">

```

### 3.7 管理AJP端口 ###

AJP是为 Tomcat 与 HTTP 服务器之间通信而定制的协议，能提供较高的通信速度和效率。如果tomcat前端放的是apache的时候，会使用到AJP这个连接器。如果用nginx做的反向代理，因此不使用此连接器，因此需要注销掉该连接器。

```
<!--
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
-->
```

### 3.8 用普通用户启动Tomcat ###

用root用户启动tomcat有一个严重的问题，那就是tomcat具有root权限。这意味着你的任何一个jsp脚本都具有root权限，所以可以轻易地用jsp脚本删除你整个硬盘里的东西！所以我们最好不要使用root启动tomcat。

权限修改见上文1.2所示。

```
#查看由哪个用户启动
ps aux | grep tomcat 
```

![](http://i.imgur.com/24sEcMq.png)


## 参考 ##

本文参考以下文章，在此对原作者表示感谢！

[Ubuntu 16.04安装Tomcat 8](http://blog.topspeedsnail.com/archives/4551)

[Tomcat安全加固配置手册](http://www.51itong.net/tomcat-4687.html)

[Tomcat 安全配置与性能优化](http://www.tuicool.com/articles/BRF732f)


